#阿里云K8S私有Ingress Controller的配置和使用
##创建集群
进入阿里云容器服务控制台，创建一个新的k8s集群，此时集群会自动生成一个公网的Ingress Controller和一个公网的SLB监听着Worker的80和443端口。

**默认的公网Ingress Controller**
![](http://ww1.sinaimg.cn/large/006tNc79gy1g4dso1aorzj31ek0em76x.jpg)

**默认的公网SLB**（SLB名字是自己起的，为了方便看）
![](http://ww1.sinaimg.cn/large/006tNc79gy1g4dshqd4cuj31ma0jawhn.jpg)
##创建私有SLB
进入负载均衡控制台创建一个私有SLB，实例规格按实际业务需求。**注意：专有网络必须和刚才创建的集群的一样！！！**

![](http://ww1.sinaimg.cn/large/006tNc79gy1g4dt4trda1j312w0u0gt4.jpg)
##配置私有Ingress Controller
由于Ingress Controller Pods中的serviceAccountName是引用集群默认创建的，所以在此就不再配置ServiceAccount、ClusterRole和ClusterRoleBinding。

1. Nginx ConfigMap
		
		apiVersion: v1
		kind: ConfigMap
		metadata:
		  name: private-nginx-configuration #名字可以自己改
		  namespace: kube-system
		  labels:
		    app: ingress-nginx
		data:
		    proxy-body-size: 20m
		    proxy-connect-timeout: "10"
		    max-worker-connections: "65536"
		    enable-underscores-in-headers: "true"
		    reuse-port: "true"
		    worker-cpu-affinity: "auto"
		    server-tokens: "false"
		    ssl-redirect: "false"
		    allow-backend-server-header: "true"
		    ignore-invalid-headers: "true"
		    generate-request-id: "true"
		    #forwarded-for-header: "X-Real-IP"
		    #compute-full-forwarded-for: "true"
		    #hsts: "false"
		    #enable-vts-status: "true"
		    #use-proxy-protocol: "true"
		---
		# nginx tcp stream config map
		kind: ConfigMap
		apiVersion: v1
		metadata:
		  name: private-tcp-services
		  namespace: kube-system
		---
		# nginx udp stream config map
		kind: ConfigMap
		apiVersion: v1
		metadata:
		  name: private-udp-services
		  namespace: kube-system
2. Nginx Ingress Controller Pods

		apiVersion: apps/v1
		kind: Deployment
		metadata:
		  name: private-nginx-ingress-controller
		  labels:
		    app: private-ingress-nginx
		  namespace: kube-system
		  annotations:
		    component.version: '0.22.0'
		    component.revision: '5'
		spec:
		  replicas: 2
		  selector:
		    matchLabels:
		      app: private-ingress-nginx
		  template:
		    metadata:
		      labels:
		        app: private-ingress-nginx
		      annotations:
		        prometheus.io/port: "10254"
		        prometheus.io/scrape: "true"
		    spec:
		      #tolerations:
		      #  - key: node-role.kubernetes.io/master
		      #    effect: NoSchedule
		      affinity:
		        podAntiAffinity:
		          preferredDuringSchedulingIgnoredDuringExecution:
		          - weight: 100
		            podAffinityTerm:
		              labelSelector:
		                matchExpressions:
		                - key: app
		                  operator: In
		                  values:
		                  - ingress-nginx
		              topologyKey: "kubernetes.io/hostname"
		      #use default serviceAccountName
		      serviceAccountName: nginx-ingress-controller
		      initContainers:
		      - name: init-sysctl
		        image: registry-vpc.cn-hongkong.aliyuncs.com/acs/busybox:latest
		        command:
		        - /bin/sh
		        - -c
		        - |
		          sysctl -w net.core.somaxconn=65535
		          sysctl -w net.ipv4.ip_local_port_range="1024 65535"
		          sysctl -w fs.file-max=1048576
		          sysctl -w fs.inotify.max_user_instances=16384
		          sysctl -w fs.inotify.max_user_watches=524288
		          sysctl -w fs.inotify.max_queued_events=16384
		        securityContext:
		          privileged: true
		      containers:
		      - name: nginx-ingress-controller
		        image: registry-vpc.cn-hongkong.aliyuncs.com/acs/aliyun-ingress-controller:v0.22.0.5-552e0db-aliyun
		        args:
		          - /nginx-ingress-controller
		          - --configmap=$(POD_NAMESPACE)/private-nginx-configuration
		          - --tcp-services-configmap=$(POD_NAMESPACE)/private-tcp-services
		          - --udp-services-configmap=$(POD_NAMESPACE)/private-udp-services
		          - --annotations-prefix=nginx.ingress.kubernetes.io
		          - --publish-service=$(POD_NAMESPACE)/private-nginx-ingress-lb
		          - --ingress-class=private #自定义名
		          - --v=2
		        env:
		          - name: POD_NAME
		            valueFrom:
		              fieldRef:
		                fieldPath: metadata.name
		          - name: POD_NAMESPACE
		            valueFrom:
		              fieldRef:
		                fieldPath: metadata.namespace
		        ports:
		        - name: http
		          containerPort: 80
		        - name: https
		          containerPort: 443
		        livenessProbe:
		          failureThreshold: 3
		          httpGet:
		            path: /healthz
		            port: 10254
		            scheme: HTTP
		          initialDelaySeconds: 10
		          periodSeconds: 10
		          successThreshold: 1
		          timeoutSeconds: 1
		        readinessProbe:
		          failureThreshold: 3
		          httpGet:
		            path: /healthz
		            port: 10254
		            scheme: HTTP
		          periodSeconds: 10
		          successThreshold: 1
		          timeoutSeconds: 1
		        securityContext:
		          capabilities:
		              drop:
		              - ALL
		              add:
		              - NET_BIND_SERVICE
		          runAsUser: 33
		        volumeMounts:
		        - name: localtime
		          mountPath: /etc/localtime
		          readOnly: true
		      nodeSelector:
		        beta.kubernetes.io/os: linux
		      volumes:
		        - name: localtime
		          hostPath:
		            path: /etc/localtime
		            type: File
3.	Nginx Ingress Service

		apiVersion: v1
		kind: Service
		metadata:
		  name: private-nginx-ingress-lb
		  namespace: kube-system
		  labels:
		    app: private-nginx-ingress-lb
		  annotations:
		    # set loadbalancer to the specified slb id
		    service.beta.kubernetes.io/alicloud-loadbalancer-id: lb-xxxx
		    # set loadbalancer address type to intranet if using private slb instance
		    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
		    service.beta.kubernetes.io/alicloud-loadbalancer-force-override-listeners: 'true'
		spec:
		  type: LoadBalancer
		  # do not route traffic to other nodes
		  # and reserve client ip for upstream
		  externalTrafficPolicy: "Local"
		  ports:
		  - port: 80
		    name: http
		    targetPort: 80
		  - port: 443
		    name: https
		    targetPort: 443
		  selector:
		    # select app=private-ingress-nginx pods
		    app: private-ingress-nginx

##部署私有Ingress Controller
	kubectl apply -f private-ingress-controller.yml
1. Private Ingress Pod
![](http://ww2.sinaimg.cn/large/006tNc79gy1g4dukry438j31eo0m4q6n.jpg)
2. Private Ingress LB Service
![](http://ww2.sinaimg.cn/large/006tNc79gy1g4dunh5yj1j31l60o6wiw.jpg)

##更新Clusterrole:nginx-ingress-controller
由于在配置私有Ingress Controller Pod时是引用集群默认的ServiceAccount，新生成的**ingress-controller-leader-private**配置项没有更新到默认的ClusterRole所以导致Service启动时会报没权限，此时我们需要在默认的ClusterRole中的resourceNames下添加**ingress-controller-leader-private**。
	
	kubectl edit clusterrole nginx-ingress-controller -o yaml
![](http://ww4.sinaimg.cn/large/006tNc79gy1g4duz5p2fjj30vu0gkmz5.jpg)

##使用阿里云DNS PrivateZone绑定SLB IP
1. 进入云解析DNS控制台，开通PrivateZone并添加域名。
![](http://ww3.sinaimg.cn/large/006tNc79gy1g4dv6598y6j328m0ni436.jpg)
2. 关联vpc
![](http://ww3.sinaimg.cn/large/006tNc79gy1g4dv7tqxqoj312c0u040p.jpg)
3. 添加域名解析，绑定私有SLB IP
![](http://ww1.sinaimg.cn/large/006tNc79gy1g4dv9wz6uhj318g0mc40g.jpg)

##部署测试服务
	apiVersion: apps/v1beta2
	kind: Deployment
	metadata:
	  annotations:
	    deployment.kubernetes.io/revision: '1'
	  generation: 1
	  labels:
	    app: demo
	  name: demo
	  namespace: default
	spec:
	  progressDeadlineSeconds: 600
	  replicas: 1
	  revisionHistoryLimit: 10
	  selector:
	    matchLabels:
	      app: demo
	  strategy:
	    rollingUpdate:
	      maxSurge: 25%
	      maxUnavailable: 25%
	    type: RollingUpdate
	  template:
	    metadata:
	      labels:
	        app: demo
	    spec:
	      containers:
	        - image: >-
	            registry-vpc.cn-hongkong.aliyuncs.com/xxxx/demo:1.0.6-1
	          imagePullPolicy: Always
	          name: demo
	          resources:
	            limits:
	              cpu: 2048m
	              memory: 4Gi
	            requests:
	              cpu: 2048m
	              memory: 4Gi
	          terminationMessagePath: /dev/termination-log
	          terminationMessagePolicy: File
	          livenessProbe:
	            httpGet:
	              path: /v1/health
	              port: 80
	            initialDelaySeconds: 180
	            periodSeconds: 10
	          readinessProbe:
	            httpGet:
	              path: /v1/health
	              port: 80
	            initialDelaySeconds: 180
	            periodSeconds: 10
	      dnsPolicy: ClusterFirst
	      restartPolicy: Always
	      schedulerName: default-scheduler
	      securityContext: {}
	      terminationGracePeriodSeconds: 300
	      
	---
	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	  annotations:
	    kubernetes.io/ingress.class: private
	    nginx.ingress.kubernetes.io/rewrite-target: /$2
	  generation: 1
	  name: private-demo-ingress
	  namespace: default
	spec:
	  tls:
	  - hosts:
	    - k8s-test.internal.abc.com
	    # 配置TLS证书
	    secretName: abc.com
	  rules:
	  - host: k8s-test.internal.abc.com
	    http:
	      paths:
	      - backend:
	          serviceName: demo-svc
	          servicePort: 80
	        path: /abc(/|$)(.*)
	        
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: demo-svc
	  namespace: default
	spec:
	  clusterIP: None
	  ports:
	    - port: 80
	      protocol: TCP
	      targetPort: 80
	  selector:
	    app: demo
	  sessionAffinity: None
	  type: ClusterIP
	  
##测试结果
在同一个VPC下的ECS访问集群服务结果如下
![](http://ww3.sinaimg.cn/large/006tNc79gy1g4dvkndhz5j31em06mgme.jpg)



