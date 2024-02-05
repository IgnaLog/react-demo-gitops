argo: installargo loginargo addcluster createns project installapp
argo-iu: installargo loginargo addcluster createns secret project installimageupdater installapp

#---------------------------------------------------------------------------- argo

installargo:
	@echo "| > INSTALACION DE ARGOCD"
	helm install argo-cd ../charts/argo-cd/ \
	--namespace argocd \
	--create-namespace --wait \
	--set server.service.type=LoadBalancer \
	-f ../charts/argo-cd/values-custom.yaml

loginargo:
	@echo "| > HACEMOS LOGIN CON ARGOCD-CLI"
    EXTERNAL_IP := $(shell kubectl get svc argo-cd-argocd-server -n argocd -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo "YES" | argocd login $(EXTERNAL_IP) --username admin --password "ARGOCDPASS"

addcluster:
	@echo "| > AÑADIMOS EL CLUSTER AKS A ARGOCD"
	yes | argocd cluster add aks-demo --name aks-demo-cluster

createns:
	@echo "| > CREACION EL NAMESPACE argocd-demo"
	kubectl create ns argocd-demo

project:
	@echo "| > CREACION DE PROYECTO DEMO"
	kubectl apply -f ../manifiests/argocd/project-argocd-demo.yaml

installapp:
	@echo "| > INSTALACION DE APLICACION DEMO"
	kubectl apply -f ../manifiests/argocd/react-demo-app.yaml

#---------------------------------------------------------------------------- argocd-iu

installargo:
	@echo "| > INSTALACION DE ARGOCD"
	helm install argo-cd ../charts/argo-cd/ \
	--namespace argocd \
	--create-namespace --wait \
	--set server.service.type=LoadBalancer \
	-f ../charts/argo-cd/values-custom.yaml

loginargo:
	@echo "| > HACEMOS LOGIN CON ARGOCD-CLI"
    EXTERNAL_IP := $(shell kubectl get svc argo-cd-argocd-server -n argocd -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo "YES" | argocd login $(EXTERNAL_IP) --username admin --password "ARGOCDPASS"

addcluster:
	@echo "| > AÑADIMOS EL CLUSTER AKS A ARGOCD"
	yes | argocd cluster add aks-demo --name aks-demo-cluster

createns:
	@echo "| > CREACION EL NAMESPACE argocd-demo"
	kubectl create ns argocd-demo

secret:
	@echo "| > INSTALACION DE GIT SECRET"
	kubectl apply -f ../manifiests/k8s-iu/secret.yaml

project:
	@echo "| > CREACION DE PROYECTO DEMO"
	kubectl apply -f ../manifiests/argocd/project-argocd-demo.yaml

installimageupdater:
	@echo "| > INSTALACION DE IMAGE UPDATER"
	kubectl apply -f ../manifiests/argocd/image-updater.yaml

installapp:
	@echo "| > INSTALACION DE APLICACION DEMO"
	kubectl apply -f ../manifiests/argocd/react-demo-app.yaml