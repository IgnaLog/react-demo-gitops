# Tutorial Jenkins - Despliegue de un Sistema de CI/CD en Microsoft Azure

¡Bienvenido al Tutorial Jenkins!

Esta guía te llevará paso a paso a través de la implementación de un robusto sistema de Integración Continua y Despliegue Continuo (CI/CD) utilizando Jenkins y la nube de Microsoft Azure. Este tutorial está especialmente diseñado para aquellos que ya cuentan con conocimientos fundamentales en Git, GitHub, Kubernetes, Trivy, SonarQube, ArgoCD, React, NPM, Node, Docker y DockerHub, así como experiencia en el manejo de la terminal de Ubuntu y entornos en la nube de Azure.

## Contenido del Tutorial

1. [Iniciando Máquinas Virtuales en Azure](#iniciando-máquinas-virtuales-en-azure)
2. [Creando Jenkins-Master VM](#creando-jenkins-master-vm)
3. [Creando Jenkins-Agent VM](#creando-jenkins-agent-vm)
4. [Creando SonarQube VM](#creando-sonarqube-vm)
5. [Generación de Claves SSH](#generación-de-claves-ssh)
6. [Accediendo a Jenkins](#accediendo-a-jenkins)
7. [Configurando Jenkins](#configurando-jenkins)
   - [Configurando Nodos](#configurando-nodos)
   - [Instalando Plugins](#instalando-plugins)
   - [Configurando Plugins](#configurando-plugins)
   - [Conectando Jenkins con SonarQube](#conectando-jenkins-con-sonarqube)
   - [Conectando Jenkins con mi Cuenta de GitHub](#conectando-jenkins-con-mi-cuenta-de-github)
   - [Conectando Jenkins con mi Cuenta de DockerHub](#conectando-jenkins-con-mi-cuenta-de-dockerhub)
8. [Creando Pipeline y Jenkinsfile](#creando-pipeline-y-jenkinsfile)
9. [Creando AKS Cluster](#creando-aks-cluster)
   - [Instalando Azure-cli (az)](#instalando-azure-cli-az)
   - [Iniciando Sesión en tu Cuenta de Azure](#iniciando-sesión-en-tu-cuenta-de-azure)
10. [Creando ArgoCD](#creando-argocd)
    - [Instalación Manual](#instalación-manual)
    - [Instalación con Helm](#instalación-con-helm)
      - [Usando Repositorio Remoto](#usando-repositorio-remoto)
      - [Usando un Chart Local](#usando-un-chart-local)
    - [Asignando Cluster AKS a ArgoCD](#asignando-cluster-aks-a-argocd)
11. [Creando Proyecto Gitops](#creando-proyecto-gitops)
    - [Conectando con el Repositorio Gitops de GitHub](#conectando-con-el-repositorio-gitops-de-github)
    - [Creando Aplicación de Monitoreo en Argocd](#creando-aplicación-de-monitoreo-en-argocd)
    - [Creando Aplicación y Proyecto en ArgoCD desde Manifiestos](#creando-aplicación-y-proyecto-en-argocd-desde-manifiestos)
    - [Creando Pipeline para CD](#creando-pipeline-para-cd)
    - [Creando Jenkinsfile para CD](#creando-jenkinsfile-para-cd)
12. [Conectando Proyecto CI con Proyecto CD](#conectando-proyecto-ci-con-proyecto-cd)
13. [Conectando Webhook de GitHub](#conectando-webhook-de-github)
14. [Conectando con Gmail](#conectando-con-gmail)
15. [Usando Image Updater](#usando-image-updater)

## Arquitectura del Sistema

La siguiente imagen representa la arquitectura del sistema que implementaremos en este tutorial.

¡Explora la imagen para comprender mejor la interacción entre estos componentes y sigue los pasos del tutorial para implementar este sistema de CI/CD en tu entorno Azure!

![](https://i.ibb.co/LYk8jcq/jenkins-schema-draw.png)

Este tutorial te llevará paso a paso desde la configuración inicial hasta la implementación de un flujo de CI/CD robusto utilizando las tecnologías mencionadas. ¡Comencemos!

## Iniciando Máquinas Virtuales en Azure

Lo primero crear tres VM con el grupo de recursos `jenkins_group`.

Una máquina virtual se llamará "Jenkins-Master", la otra "Jenkins-Agent" y la última máquina virtual se llamará "SonarQube".

En Jenkins-Master abrimos el puerto 8080 TCP con el nombre JenkinsPort.

En SonarQube abrimos el puerto TCP 9000.

## Creando Jenkins-Master VM

- Crear una VM DS1_v2 con 1CPUs y 3.5GiB de memoria RAM y 15GB o más de almacenamiento.

```
sudo chmod 700 Jenkins-Master_key.pem
```

```
ssh -i <Key-pair .pem> azureuser@<Public IP>
```

```
sudo apt update
```

```
sudo apt upgrade
```

```
sudo init 6
```

```
sudo apt install openjdk-17-jre-headless
```

```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

```
sudo systemctl enable jenkins
```

```
sudo systemctl start jenkins
```

```
systemctl status jenkins
```

## Creando Jenkins-Agent VM

- Crear una VM DS1_v2 con 1CPUs y 3.5GiB de memoria RAM y 15GB o más de almacenamiento.

```
sudo chmod 700 Jenkins-Agent_key.pem
```

```
ssh -i <Key-pair .pem> azureuser@<Public IP>
```

```
sudo init 6
```

```
sudo apt install openjdk-17-jre-headless
```

```
sudo apt-get install docker.io
```

```
sudo usermod -aG docker $USER
```

```
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

```

```
sudo init 6
```

## Creando SonarQube VM

- Crear una VM B2s con 2CPUs y 4GiB de memoria RAM y 15GB o más de almacenamiento.

- Habilitarle el ppuerto TCP 9000.

- Instalar y configurar una base de datos postgresql:

```
sudo apt update
sudo apt upgrade
```

```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

```
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
```

```
sudo apt update
```

```
sudo apt-get -y install postgresql postgresql-contrib
```

```
sudo systemctl enable postgresql
```

```
sudo passwd postgres
```

```
su - postgres
```

```
createuser sonar
```

```
psql
```

```
ALTER USER sonar WITH ENCRYPTED password 'sonar';
```

```
CREATE DATABASE sonarqube OWNER sonar;
```

```
grant all privileges on DATABASE sonarqube to sonar;
```

```
\q
```

```
exit
```

- Agregar el repositorio de Adaptium:

```
sudo bash
```

```
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc

```

```
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
```

```
apt install temurin-17-jdk
```

```
update-alternatives --config java
```

```
/usr/bin/java --version
```

```
exit
```

- Modificamos el kernel de Linux.

  - Aumentar límites:

  ```
  sudo vim /etc/security/limits.conf
  ```

  - Pega los valores que se encuentran a continuación al final del archivo:

    ```
    sonarqube - nofile 65536
    sonarqube - nproc 4096
    ```

  - Aumentar las regiones de memoria mapeada:

  ```
  sudo vim /etc/sysctl.conf
  ```

  - Pegar los siguientes valores al final del archivo:
    ```
    vm.max_map_count = 262144
    ```

  ```
  sudo init 6
  ```

- Instalamos SonarQube.

  - Descargar y extraer:
    ```
    sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
    sudo apt install unzip
    sudo unzip sonarqube-9.9.0.65466.zip -d /opt
    sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube
    ```
  - Crear usuario y establecer permisos:
    ```
    sudo groupadd sonar
    sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
    sudo chown sonar:sonar /opt/sonarqube -R
    ```
  - Actualizar las propiedades de Sonarqube con las credenciales de la base de datos:
    ```
    sudo vim /opt/sonarqube/conf/sonar.properties
    ```
  - Encontrar y reemplazar los siguientes valores, es posible que necesite agregar el sonar.jdbc.url:
    ```
    sonar.jdbc.username=sonar
    sonar.jdbc.password=sonar
    sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
    ```
  - Crear servicio para Sonarqube:

    ```
    sudo vim /etc/systemd/system/sonar.service
    ```

    - Pegar lo siguiente en el archivo:

    ```
    [Unit]
    Description=SonarQube service
    After=syslog.target network.target

    [Service]
    Type=forking

    ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
    ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

    User=sonar
    Group=sonar
    Restart=always

    LimitNOFILE=65536
    LimitNPROC=4096

    [Install]
    WantedBy=multi-user.target
    ```

- Iniciar Sonarqube y habilitar el servicio:

  ```
  sudo systemctl start sonar
  sudo systemctl enable sonar
  sudo systemctl status sonar
  ```

- Observar archivos de registro y monitorizar el inicio:

  ```
  sudo tail -f /opt/sonarqube/logs/sonar.log
  ```

- Accedemos a SonarQube:
  - Accedemos con el navegador a: `<Public IP VM Sonar-Scanner>:9000`.
  - User -> admin, Password -> admin.

## Generación de Claves SSH

- Modificar la configuración del servidor ssh en ambas VM: jenkins-master y jenkins-agent.

  ```
  sudo nano /etc/ssh/sshd_config
  ```

  Descomentar:

  ```
  PubkeyAuthentication yes
  AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
  ```

  Reinicar el servidor ssh:

  ```
  sudo service sshd reload
  ```

- Generar claves.

  En jenkins-master ejecutar:

  ```
  ssh-keygen
  ```

  ```
  cd ./ssh
  ```

  ```
  cat id_rsa.pub
  ```

  Copiar la clave en jenkins-agent ./ssh/authorized_keys dejando un párrafo entre clave y clave:

  ```
  cd .ssh/
  ```

  ```
  sudo nano authorized_keys
  ```

## Accediendo a Jenkins

Entra en jenkins desde el navegador con la ip pública de la VM más el puerto 8080 `<Public IP>:8080`.

Accede a la clave de acceso de jenkins:

```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Configurando Jenkins

### Configurando Nodos

Poner el nodo maestro con 0 número de ejecutores.

Crear un nuevo nodo con jenkins-agent:

- Poner este nuevo nodo con 2 números de ejecutores.
- Directorio: averiguarlo con `pwd` en la vm agente -> `/home/azureuser`.
- Que se ejecute siempre que sea posible.
- Conectarlo con ssh a la vm jenkins-agent con su public ip y añadiendo la clave privada que generamos en la vm maestra.
- Host Key Verification Strategy -> Non Verifying Verification Strategy.
- Probar que se ejecuta en el nodo agente haciendo un test pipeline.

### Instalando Plugins

Buscamos los siguientes plugins:

- sonarqube -> SonarQube Scanner.
- gates -> Sonnar Quality Gates + Quality Gates.
- nodejs -> NodeJS.
- docker -> Docker + Docker Commons + Docker Pipeline + Docker API + docker-build-step + CloudBees Docker Build and Publish.

### Configurando Plugins

Dentro del apartado Tools de Jenkins:

- Instalaciones de NodeJS:

  - Nombre -> node20.
  - Seleccionamos la última versión de nodejs LTS que haya en la web o la que tengamos usando en nuestro proyecto.

- Instalaciones de Docker:

  - Nombre -> docker.
  - Instalar automáticamente -> añadir un instalador -> download from docker -> latest.

- Instalaciones de SonarQube Scanner:
  - Nombre -> sonar-scanner.
  - Instalar automáticamente -> Install from Maven Central -> SonarQube Scanner 5.0.1.3006.
  - Para quitar los warnings de sonarqube en jenkins entramos en: Manage Jenkins -> Security -> Hidden security warnings -> desactivamos Security warnings.

### Conectando Jenkins con SonarQube

- Conectando Jenkins con sonarqube:

  - Creamos el token para jenkins:

    En SonarQube -> My Account -> Security:

    ```
    nombre: for-jenkins-token
    type: Global Analysis Token
    expiration: no expiration
    ```

    En Jenkins añadimos una credencial del tipo secret-text con ese token.

- Añadiendo SonarQube al sistema de Jenkins:

  En Manage Jenkins -> System -> SonarQube Servers -> add sonarqube:

  - Name: Sonarqube-Server
  - Url del servidor: `http://<Public IP sonarqube VM>:9000`

- Añadiendo un quality-gate para sonarqube:

  - En sonarqube: Quality-Gates -> Create.

    ```
    Name: SonarQube-Quality-Gate
    ```

- Añadiendo un webhook de SonarQube para Jenkins:

  - En sonarqube: Administration -> Configuration -> Webhooks -> Create.

    ```
    name: for-jenkins-webhook
    URL: http://<Public IP Jenkins-Master VM>:8080/sonarqube-webhook/
    ```

- Añadiendo un proyecto a sonarqube:

  - En sonarqube: Manually.
    ```
    Project display name: React-Demo-Ci
    ```
    Setup -> Locally -> 30 days -> Generate -> Continue -> Other -> Linux.

### Conectando Jenkins con mi Cuenta de GitHub

En la seccion de credenciales añadir una nueva credencial:

- Será de tipo username and password.
- Poner username de github con mayus.
- En Github -> Settings -> Developer Settings -> Personal Access Tokens -> Tokens (Classic) -> Generate new classic token -> Darle todos los permisos sin expiración.

### Conectando Jenkins con mi Cuenta de DockerHub

En dockerhub -> My Account -> Security -> New Access Token
Name: for-jenkins-token.

En jenkins: Creamos una credencial del tipo username and password.

## Creando Pipeline y Jenkinsfile

Crear Pipeline:

- Crear un job pipeline llamado React-Demo-CI.
- Marcar Desechar Ejecuciones Antiguas.
- Número máximo de ejecuciones para guardar -> 2.
- Pipeline -> Pipeline script from SCM.
- SCM -> Git.
- Poner credenciales.

Crear Jenkinsfile:

```
pipeline {
    agent any
    tools {
        jdk "jdk17"
        nodejs "node20"
    }
    environment {
        SCANNER_HOME = tool "sonar-scanner"
        APP_NAME = "react-demo-ci"
        RELEASE = "0.1.0"
        DOCKER_USER = "ignalog"
        DOCKER_PASS = "dockerhub"
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }
    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }
        stage("Checkout form GitHub") {
            steps {
                git branch: "main", url: "https://github.com/IgnaLog/react-demo-ci"
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('Sonarqube-Server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=React-Demo-CI -Dsonar.projectKey=React-Demo-CI'''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: "SonarQube-Token"
                }
            }
        }
        stage("Install Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage("Trivy Fs Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Testing") {
            steps {
                sh "npm run test"
            }
        }
        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage("Trivy Image Scan") {
            steps {
                script {
                    sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ignalog/react-demo-ci:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt')
                }
            }
        }
        stage ("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }
    }
}
```

```
sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ignalog/react-demo-ci:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt')
```

Este comando ejecuta un contenedor Docker utilizando la imagen `aquasec/trivy` y escanea la imagen `ignalog/react-demo-ci:latest:latest`. Aquí está el significado de los parámetros utilizados:

- `-v /var/run/docker.sock:/var/run/docker.sock`: Monta el socket de Docker dentro del contenedor, lo que permite al contenedor interactuar con el demonio de Docker en el host.
- `--no-progress`: Deshabilita la barra de progreso para el escaneo.
- `--scanners vuln`: Indica que se deben utilizar solo los escáneres de vulnerabilidades.
- `--exit-code 0`: Establece el código de salida del contenedor como 0, independientemente de si se encuentran vulnerabilidades o no. Esto puede ser útil para evitar que el pipeline falle solo por el resultado del escaneo.
- `--severity HIGH,CRITICAL`: Especifica las severidades de las vulnerabilidades que se deben incluir en el informe.
- `--format table`: Indica el formato del informe de escaneo como una tabla.
- `> trivyimage.txt`: Redirige la salida del comando a un archivo llamado trivyimage.txt.

## Creando AKS Cluster

- Creamos el grupo de recursos aks_group.
- Lo configuramos como Desarrollo/pruebas.
- Nombre del cluster: aks-demo.
- Region: Germany Central (Si hemos cumplido el límite de núcleos de cpu para France Central, seleccionamos Germany Central porque sino al revisar nos dará error y habría que solicitar más núcleos para esa zona).
- Zona de disponibilidad: Ninguno.
- Tarifa: Gratis.
- Versión: última disponible.
- Actualización: Habilitado con revisión.
- Autenticación y autorización: Cuentas locales con RBAC de Kubernetes.
- Elegimos un D2s_v3 con 2cpu, 8GiB de memoria.
- Escalabilidad automática.
- Mínimo de nodos: 2
- Máximo de nodos: 5
- Supervisión: Desactivado.
- El resto todo predeterminado.

### Instalando Azure-cli (az)

```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Iniciando Sesión en tu Cuenta de Azure

Se recomienda utilizar un navegador en el que ya haya iniciado sesión en su portal de Azure:

```
az login
```

Si tiene problemas, intente usar:

```
az login --use-device-code
```

Establecer la suscripción del clúster:

```
az account set --subscription <Id. de suscripción>
```

Descargar las credenciales del clúster:

```
az aks get-credentials --resource-group aks_group --name aks-demo --overwrite-existing
```

Verificar que estamos conectados al clúster de AKS:

```
kubectl get nodes
```

## Creando ArgoCD

---

### Requisitos

- Tener instalado el binario de Helm. https://helm.sh/docs/intro/install/
- Tener instalado kubectl.

---

### Instalación Manual

```
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Instalamos argocd-cli:

```
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```

Ahora tenemos que hacer que el servicio de argocd-server sea LoadBalancer para poder acceder a el desde el exterior, para ello ejecutamos:

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Ahora visualizamos los servicios:

```
kubectl get svc -n argocd
```

Deberíamos tener el servicio argocd-server como LoadBalancer con su external-ip asignada con la que usaremos para acceder a argocd desde el navegador:

| NAME          | TYPE         | CLUSTER-IP   | EXTERNAL-IP | PORT(S)                    | AGE   |
| ------------- | ------------ | ------------ | ----------- | -------------------------- | ----- |
| argocd-server | LoadBalancer | 10.0.250.234 | 51.103.5.91 | 80:31340/TCP,443:30142/TCP | 8m59s |

Una vez accedido desde el navegador a argocd con la ip `51.103.5.91` ponemos como usuario admin y contraseña la sacamos escribiendo en consola:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Esta será la contaseña que usaremos para acceder por primera vez a argocd. Después desde la interfaz podemos cambiarla en el apartado User Info > UPDATE PASSWORD.

Ahora hacemos login con argocd-cli para poder usar el comando argocd:

```
argocd login <external ip> --username admin
```

### Instalación con Helm

#### Usando Repositorio Remoto

Añadimos el repo de Helm:

```
helm repo add argo https://argoproj.github.io/argo-helm
```

Hacemos pull del Chart para descargarlo, poder ver el contenido del Chart e instalarlo:

```
helm pull argo/argo-cd
```

_Añade si deseas `--version 5.8.2` para tener la misma versión de mi repo gitops, sino se descargará la última versión_

Descomprimimos el paquete TGZ del Chart descargado:

```
tar -zxvf argo-cd-5.8.2.tgz
```

Hacemos la instalación pasando como parámetro el repositorio gitops al que va hacer referencia:

```
helm install argo-cd argo-cd/ \
  --namespace argocd \
  --create-namespace --wait \
  --set configs.credentialTemplates.github.url=https://github.com/IgnaLog
```

Si queremos una instalación a un repositorio de GitHub privado tenemos que crear un token de acceso en la cuenta de GitHub y guardarlo en: `~/.secrets/github/ignalog/token`. Además crear un fichero con el nombre de usuario del repositorio y guardarlo en: `~/.secrets/github/ignalog/user`. Después ejecutamos:

```
helm install argo-cd argo-cd/ \
  --namespace argocd \
  --create-namespace --wait \
  --set configs.credentialTemplates.github.url=https://github.com/IgnaLog \
  --set configs.credentialTemplates.github.username=$(cat ~/.secrets/github/ignalog/user) \
  --set configs.credentialTemplates.github.password=$(cat ~/.secrets/github/ignalog/token)
```

Levantamos un Port-Forward para poder acceder a ArgoCD UI desde localhost:9090.

```
kubectl port-forward service/argo-cd-argocd-server -n argocd 9090:443
```

Tambien podemos hacer que el servicio de argo-cd-argocd-server sea LoadBalancer para poder acceder a él desde el exterior sin tener que levantar un port-forward, para ello ejecutamos:

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Tambien puede hacer una instalación con argo-cd-argocd-server como LoadBalancer sin tener que hacer estos pasos anteriores ejecutando:

```
helm install argo-cd argo-cd/ \
  --namespace argocd \
  --create-namespace --wait \
  --set server.service.type=LoadBalancer \
  --set configs.credentialTemplates.github.url=https://github.com/IgnaLog \
  --set configs.credentialTemplates.github.username=$(cat ~/.secrets/github/ignalog/user) \
  --set configs.credentialTemplates.github.password=$(cat ~/.secrets/github/ignalog/token)
```

Ahora visualizamos los servicios:

```
kubectl get svc -n argocd
```

Deberíamos tener el servicio argocd-server como LoadBalancer con su external-ip asignada con la que usaremos para acceder a argocd desde el navegador:

| NAME                  | TYPE         | CLUSTER-IP   | EXTERNAL-IP | PORT(S)                    | AGE   |
| --------------------- | ------------ | ------------ | ----------- | -------------------------- | ----- |
| argo-cd-argocd-server | LoadBalancer | 10.0.250.234 | 51.103.5.91 | 80:31340/TCP,443:30142/TCP | 8m59s |

Una vez accedido desde el navegador a argocd con la ip externa `51.103.5.91` o través de `localhost:9090` con el port-forward iniciamos sesión en argocd por primera vez. Ponemos como usuario admin y contraseña la obtenemos escribiendo en la terminal:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Esta será la contaseña que usaremos para acceder por primera vez a argocd. Después desde la interfaz podemos cambiarla en el apartado: User Info -> UPDATE PASSWORD.

Ahora hacemos login con argocd-cli para poder usar el comando argocd:

```
$argocd login <external ip> --username admin
o
$argocd login localhost:9090
```

Tambien podemos cambiar la contraseña desde la terminal:

```
argocd account update-password
```

#### Usando un Chart Local

Necesitamos tener descargado en local todo el proyecto de "react-demo-gitops" donde se encuentra el repositorio de argo-cd.

Tenemos que modificar el _values-custom.yaml_ de _charts/argo-cd_ para que se adape a nuestro repositorio gitops:

```
  credentialTemplates:
    github-user-public:
      url: https://github.com/IgnaLog
```

```
  repositories:
    ignalog-react-demo-gitops:
      url: https://github.com/IgnaLog/react-demo-gitops.git
      name: ignalog-react-demo-gitops
```

Sustiruir esos valores por los de nuestro repositorio.

Desde ese repositorio gitops en local nos movemos al directorio scripts y ejecutamos lo siguiente:

```
helm install argo-cd ../charts/argo-cd/ \
  --namespace argocd \
  --create-namespace --wait \
  -f ../charts/argo-cd/values-custom.yaml
```

Ahora para acceder a argocd levantamos un port-forward:

```
kubectl port-forward service/argo-cd-argocd-server -n argocd 9090:443
```

O tambien podemos hacer una instalación en el que el service sea LoadBalancer y acceder desde la ip externa:

```
helm install argo-cd ../charts/argo-cd/ \
  --namespace argocd \
  --create-namespace --wait \
   --set server.service.type=LoadBalancer \
  -f ../charts/argo-cd/values-custom.yaml
```

```
kubectl get svc -n argocd
```

La contraseña la tengo establecida como `ARGOCDPASS` en mi _values-custom.yaml._

Ahora hacemos login con argocd-cli para poder usar el comando argocd:

```
argocd login <external ip> --username admin
```

O si levantamos un port-forward, utilizar:

```
argocd login localhost:9090
```

### Asignando Cluster AKS a ArgoCD

---

### Requisitos

- Tener instalado argocd-cli.
- Haberse logeado correactamente con argcd-cli.

---

Después tenemos que crear el cluster aks en argocd, para ello podemos ver los clusters de argocdc con:

```
argocd cluster list
```

Aparecerá por defecto el cluster: https://kubernetes.default.svc

Para añadir el cluster aks primero vemos como se llama ejecutando:

```
  kubectl config get-contexts
```

Vemos que tengo dos cluster, uno partiular de minikube y el de aks:

| CURRENT NAME | CLUSTER  | AUTHINFO                                  | NAMESPACE |
| ------------ | -------- | ----------------------------------------- | --------- |
| \*           | aks-demo | aks-demo clusterUser_rg-aks-demo_aks-demo | default   |
|              | minikube | minikube                                  | default   |

Para crear el cluster aks en argocd ejecutamos:

```
argocd cluster add <nombre del cluster (aks-demo)> --name <nombre que se mostrará en argocd (aks-demo-cluster)>
```

Ejecutamos `arcocd clustter list` para ver de nuevo que se ha creado el cluster en argocd.

## Creando Proyecto Gitops

### Conectando con el Repositorio Gitops de GitHub

_Sólo si NO hemos añadido el repo de gitops que vamos a usar en la instalación._

Dentro de la interfaz de ArgoCD: Settings -> + Connect Repo.

- Si es un proyecto privado: via SSH y si no usamos via HTTPS.
- Project: Default.
- Repository URL: La URL del repo gitops de GitHub.
- Username: Usuario de GitHub.
- Password: Token de acceso a GitHub (Solo si el repo es privado).

_Tambien se puede hacer desde comandos de argocd-cli._

### Creando Aplicación de Monitoreo en Argocd

En Applications -> New App.

- Application Name: react-demo-ci.
- Project Name: default.
- Sync policy: Automatic.
  - Marcamos Prune Resources.
  - Marcamos Self Heal.
- En Source:
  - Repository URL: URL del repo gitops de GitHub.
  - Revision: HEAD.
  - Path: Ruta a los manifiestos de k8s. En mi caso `manifiests/k8s`.
- En Destionation:
  - Cluster URL: Elegimos la URL del cluster AKS.
  - Namespace: default.
- Create.

_Tambien se puede hacer desde comandos de argocd-cli o desde un manifiest ".yaml" del tipo application._

### Creando Aplicación y Proyecto en ArgoCD desde Manifiestos

_Recomendado usar este método para ahorrar tiempo._

- En manifiests/argocd/project-argocd-demo.yaml es un manifiesto para configurar un proyecto en argocd. Cambiar los siguientes parámetros:

  - `spec.destination.server: https://aks-demo-dns-t5gq24d4.hcp.germanywestcentral.azmk8s.io:443` -> Cambiar por la dirección del cluster de kubernetes.
  - `spec.destination.name: aks-demo-cluster` -> Poner el nombre que queremos que tenga ese cluster en argocd.

- En manifiests/argocd/react-demo-app.yaml es un manifiesto para crear una aplicación en argocd:

  - `metadata.namespace:argocd` -> En que namespace se va ejecutar.
  - `metadata.annotations:` -> Comentar todas la anotaciones si no vamos a usar image-updater.

  - `spec.project: argocd-demo` -> Sustituimos por el nombre del proyecto creado.
  - `spec.source.repoURL: https://github.com/IgnaLog/react-demo-gitops.git` Cambiar a nuestro repositorio gitops.
  - `spec.source.targetRevision: main` -> La rama de git a revisar.
  - `spec.source.path: manifiests/k8s` -> Ubicación de los manifiestos de kubernetes que va aplicar.
  - `spec.source.destination.destination.server: https://aks-demo-dns-t5gq24d4.hcp.germanywestcentral.azmk8s.io:443` -> Cambiar a la dirección de nuestro servidor del cluster de kubernetes.
  - `spec.source.destination.destination.namespace: argocd` -> En que namespace se va ejecutar.

### Creando Pipeline para CD

Creamos un job pipeline llamado React-Demo-CD.

- Marcamos discard old builds -> max of builds to keep: 2.
- Marcamos This project is parameterized -> Add parameter -> String parameter.
  - Name: IMAGE_TAG.
- Build Triggers:
  - Marcamos Trigger builds remotely.
    - En Authentication Token: gitops-token.
- En Pipeline -> Pipeline script from SCM -> SCM -> Git -> Colocamos la URL del repo gitops de github y el token de GitHub -> rama main.

### Creando Jenkinsfile para CD

```
pipeline {
  agent { label "Jenkins-Agent" }
  environment {
    APP_NAME = "react-demo-ci"
  }
  stages {
    stage("Cleanup Workspace") {
      steps {
        cleanWs()
      }
    }
    stage("Checkout from SCM") {
      steps {
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/IgnaLog/react-demo-gitops/tree/main/manifiests/k8s'
      }
    }
    stage("Update the Deployment Tags") {
      steps {
        sh """
          cat deployment.yaml
          sed -i 's/${APP_NAME}.*/${APP_NAME}:${IMAGE_TAG}/g' deployment.yaml
          cat deployment.yaml
        """
      }
    }
    stage("Push the changed deployment file to Git") {
      steps {
        sh """
          git config --global user.name "IgnaLog"
          git config --global user.email "ignacio.coding@gmail.com"
          git add deployment.yaml
          git commit -m "Updated Deployment Manifest"
        """
        withCredentials([gitUsernamePassword(credentialsId: 'GitHub-Token', gitToolName: 'Default')]) {
          sh "git push https://github.com/IgnaLog/react-demo-gitops/tree/main/manifiests/k8s main"
        }
      }
    }
  }
}
```

Cambia la url `https://github.com/IgnaLog/react-demo-gitops/tree/main/manifiests/k8s` por la url donde se encuentren los manifiestos de kubernetes en tu repo gitops.

## Conectando Proyecto CI con Proyecto CD

- En nuestro usuario admin dentro de Jenkins ->
- Selecionamos configure ->
- En API Token ->
- Add new token ->
- JENKINS_API_TOKEN ->
- Lo generamos y lo copiamos el token ->
- Guardamos ->
- Nos dirigimos a Manage Jenkins ->
- Credentials ->
- New Credential ->
- Seleccionamos un credencial del tipo "secret text" ->
- Pegamos el token JENKINS_API_TOKEN ->
- Lo nombramos como JENKINS_API_TOKEN ->
- Create.

Ahora hay que modificar el Jenkinsfile de _react-demo-ci_ pipeline para que pueda conectarse con el Jenkinsfile del CD _react-demo-gitops_:

En environment de CI introducimos:

```
JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
```

Este es el token de jenkins que nos servirá para conectar con el pipeline CD.

Ahora tenemos que añadir este escenario en el Jenkinsfile CI:

```
stage("Trigger CD Pipeline") {
  steps {
    script {
      sh "curl -v -k --user admin:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' '20.199.113.26:8080/job/React-Demo-CD/buildWithParameters?token=gitops-token'"
    }
  }
}
```

`admin:${JENKINS_API_TOKEN}` -> Sustituir por el usuario de Jenkins y por el nombre del token que hemos creado en Jenkins para acceder desde otro job.

`20.199.113.26:8080/job/React-Demo-CD/` -> Sustituir por la url al entrar en el job de _React-Demo-CD_ de Jenkins. Después añadirle el token creado como parámetro: `buildWithParameters?token=gitops-token`

## Conectando Webhook de GitHub

Permitirá ejecutar automáticamente el pipeline por cada commit que haya en nuestro repositorio CI.

Ir al CI job:

- Configurar ->
  - Build Triggers ->
    - GitHub hook trigger from GITScm polling.
  - GitHub project ->
    - Project url: Url del proyecto CI.
- Save.

- Ir a setting del proyecto CI (react-demo-ci) de github ->
- Webhooks ->
- Add webhook - Payload URL: `http://<Public IP Jenkins server>:8080/github-webhook/`

Ahora para recibir un status de si ese commit fue desplegado con éxito por jenkins necesitamos añadir la siguientes líneas al final de nustro Jenkinsfile CI:

```
    post {
        success {
            script {
                updateGitHubCommitStatus('success')
            }
        }
        failure {
            script {
                updateGitHubCommitStatus('failure')
            }
        }
    }


def updateGitHubCommitStatus(String status) {
    def gitHubCredentialsId = "GitHub-Token"
    def commitSha = sh(script: "git rev-parse HEAD", returnStdout: true).trim()

    withCredentials([usernamePassword(credentialsId: gitHubCredentialsId, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        script {
            sh """
                curl -X POST \
                -u $USERNAME:$PASSWORD \
                -H 'Accept: application/vnd.github.v3+json' \
                https://api.github.com/repos/IgnaLog/react-demo-ci/statuses/$commitSha \
                -d '{"state":"$status", "context":"Jenkins"}'
            """
        }
    }
}
```

_Sustituye la variable gitHubCredentialsId por tu token de GitHub._

## Conectando con Gmail

Esto te permitirá enviarte un gmail cuando tu proyecto de CI se haya construido con éxito, tambien lo puedes usar en el jenkins de CD para saber cuándo ha sido desplegado. En él puedes incluir los informes de Trivy.

_Sólo podemos hacer esto si tenemos la verificación en dos pasos en nuestra cuenta de Google._

- Nos dirijimos a gestionar tu cuenta de Google ->
- Security ->
- Verificación en dos pasos ->
- Vamos al final de la página en Constraseñas de aplicación ->
  - Name: `React-demo` ->
  - Create ->
  - Copy the password ->
- Ir a Jenkins Credentials ->
- Añadir una username-password credencial ->
  - Username: Nuestro email.
  - Password: Password de gmail.
  - ID: Gmail-Token.

Ir a Manage Jenkins -> System -> Email Notification ->

- smtp server: `smtp.gmail.com`.
- Default user email suffix: `ignacio.coding@gmail.com`.
- Advanced:
  - Use smtp authentication.
    - username: `ignacio.coding@gmail.com`.
    - password: "password de gmail que hemos creado".
  - Marcamos use SSL.
  - smtp port: `465`.
- Hacemos un test para ver si funciona.

Ir a Extended Email Notification:

- smtp server: `smtp.gmail.com`.
- port: `465`.
- Advanced:
  - Añadimos la credencial de Gmail-Token.
  - Marcamos use SSL.
- Default user email suffix: `ignacio.coding@gmail.com`.
- Default content type: HTML (text/html).
- Default triggers (activadores predeterminados): Always, Success, Failure (Any).

Apply Save.

Ahora vamos añadir esto al final de nuestro Jenkinsfile:

```
post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'ignacio.coding@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
    }
}
```

Pero si tenemos configurado el paso anterior para conectar con los commit de GitHub, como solo poder tener una sentencia post en nuestro pipeline, quedaría de la siguiente manera:

```
post {
    success {
        script {
            updateGitHubCommitStatus('success')
        }
    }
    failure {
        script {
            updateGitHubCommitStatus('failure')
        }
    }
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'ignacio.coding@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
    }
}
```

_Sustituir el email por el que configuramos._

## Usando Image Updater

Este plugin de Argocd permite detectar nuevas imágenes en repositorios como Docker Hub, GitHub Container Registry u otros sin tener que tener un job CD en Jenkins que actualice a las nuevas versiones de las imágenes. De ésta manera nos ahorramos mucho tiempo y tareas.

En la siguiente ilustración, podemos observar la simplificación de nuestra arquitectura CI/CD y cómo hemos eliminado el Job "React-Demo-CD" de la misma:

![](https://i.ibb.co/gD6ywk4/jenkins-iu-schema-draw.png)

Pasos a seguir para configurar el repositorio gitops de GitHub y aprovechar las funcionalidades de Image Updater:

- En el repo de _charts/argocd-image-updater_ hay un _values-custom.yaml_ para personalizar. Aquí es necesario cambiar la etiqueta _config.registires_ para seleccionar desde que repositorio en la nube vamos a buscar la nueva imagen. Puede ser desde DockerHub, GiHub Container, etc.

- En _manifiests/argocd/image-updater.yaml_ es un manifiesto de configuración de la aplicación image-updater para nuestro argocd. Aquí hay que adaptar las siguientes etiquetas:

  - `spec.project: argocd-demo` -> Cambiar al nombre de proyecto que tenemos configurado.
  - `spec.source.repoURL: https://github.com/IgnaLog/react-demo-gitops.git` Cambiar a nuestro repositorio gitops.
  - `spec.source.targetRevision: main` -> La rama git a revisar.
  - `spec.source.path: charts/argocd-image-updater` -> Ubicación del repo de helm de image-updater.
  - `spec.source.helm.valueFiles.values-custom.yaml` -> Nombre de los values que queremos que use image-updater.
  - `spec.source.destination.destination.server: https://aks-demo-dns-t5gq24d4.hcp.germanywestcentral.azmk8s.io:443` -> Cambiar a la dirección de nuestro servidor del cluster de kubernetes.
  - `spec.source.destination.destination.namespace: argocd` -> En que namespace se va ejecutar.

- En _manifiests/argocd/project-argocd-demo.yaml_ es un manifiesto para configurar un proyecto en argocd. Cambiar los siguientes parámetros:

  - `spec.destination.server: https://aks-demo-dns-t5gq24d4.hcp.germanywestcentral.azmk8s.io:443` -> Cambiar por la dirección del cluster de kubernetes
  - `spec.destination.name: aks-demo-cluster` -> Poner el nombre que queremos que tenga ese cluster en argocd

- En _manifiests/argocd/react-demo-app.yaml_ es un manifiesto para crear una aplicación en argocd:

  - `metadata.namespace:argocd` -> En que namespace se va ejecutar
  - `metadata.annotations:`

```
argocd-image-updater.argoproj.io/image-list: image=docker.io/ignalog/react-demo-ci
argocd-image-updater.argoproj.io/image.update-strategy: latest
argocd-image-updater.argoproj.io/image.allow-tags: regexp:^0.1.0-[0-9]+$
```

Estos parámetros sirven para actualizar la última compilación subida en la versión 0.1.0 que hemos establecido.

- `docker.io` -> Es el repositorio de imágenes Dockerhub donde se encutra nuestra imagen. Permanecerá analizando por nuevos cambios. Tambien podría utilizar un repositorio de imágenes de GitHub. En este caso poner `ghcr.io`.
- `ignalog` -> Es el nombre de usuario de dockerhub.
- `react-demi-ci`: Es el nombre de la imagen.
- `image` -> Definir un alias con los tres parámetros anteriores.
- `image.update-strategy: latest` -> Esta opción hará que vigile por la última versión disponible en la siguiente expresión regular:
- `image.allow-tags: regexp:^0.1.0-[0-9]+$` -> Controla cualquier compilación numérica para la versión 0.1.0

Considere usar la siguiente estrategia si obtiene límites de uso en dockerhub:

```
argocd-image-updater.argoproj.io/image-list: yourtool=docker.io/ignalog/react-demo-ci:latest
argocd-image-updater.argoproj.io/yourtool.update-strategy: digest
```

Considere cambiar su estrategia CI cambiando el número de versión sin compilación y use patrones semVer:

```
argocd-image-updater.argoproj.io/image-list: docker.io/ignalog/react-demo-ci:~0.1
```

- `~0.1` Se refiere a que versiones que va estar revisando. En este caso todas las versiones patch de la version 0.1. Este parámetro se puede cambiar por según: https://github.com/Masterminds/semver#checking-version-constraints

  _Más información en https://argocd-image-updater.readthedocs.io/en/stable/configuration/images/_

- `spec.project: argocd-demo` -> Sustituimos por el nombre del proyecto creado.
- `spec.source.repoURL: https://github.com/IgnaLog/react-demo-gitops.git` Cambiar a nuestro repositorio gitops.
- `spec.source.targetRevision: main` -> La rama de git a revisar.
- `spec.source.path: manifiests/k8s-iu` -> Ubicación de los manifiestos de kubernetes que va aplicar.
- `spec.source.destination.destination.server: https://aks-demo-dns-t5gq24d4.hcp.germanywestcentral.azmk8s.io:443` -> Cambiar a la dirección de nuestro servidor del cluster de kubernetes.
- `spec.source.destination.destination.namespace: argocd` -> En que namespace se va ejecutar.

- En _k8s-iu/deployment.yaml_:

  - `revisionHistoryLimit: 1` -> Determina el número de replicasets que se quedarán activos sin pods una vez que haya una nueva versión. En este caso solo habrá un replicaset sin pod al cargar nuevas versiones.
  - `image: docker.io/ignalog/react-demo-ci` -> No hay que especificar la versión en el nombre de la imagen.

- En _k8s-iu/secret.yaml_:
  - `metadata.name` -> Poner el nombre del token creado en dockerhub.
  - `data.creds` -> Crearlo con `echo -n "username:password" | base64` donde username es el nombre de usuario de dockerhub y password es el token creado.
