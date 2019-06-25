kubectl edit clusterrole nginx-ingress-controller -o yaml

kubectl get clusterrole nginx-ingress-controller -o yaml

ingress-controller-leader-private

ali-aws-cloud-secret-test

#download template.yml
wget https://acs-k8s-ingress.oss-cn-hangzhou.aliyuncs.com/ingress-controller-template.yml.j2

#install jinja2-cli
sudo pip install jinja2-cli

#create ingress-controller.yml
jinja2 -D Namespace='kube-system' -D LoadbalancerID='Your private slb id' -D IngressClass='private' ingress-controller-template.yml.j2 > ingress-controller.yml