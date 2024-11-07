# Despliegue de WordPress utilizando MicroK8s en IaaS EC2 (Proyecto 2)

## Estudiante(s): 
Jonathan Betancur, jbetancur3@eafit.edu.co  
Esteban Vergara Giraldo, evergarag@eafit.edu.co  
Miguel Angel Cock Cano, macockc@eafit.edu.co

## Profesor: 
Alvaro Enrique Ospina SanJuan, aeospinas@eafit.edu.co  

---

## 1. Breve descripción

En este proyecto, desplegamos la aplicación WordPress en un clúster de alta disponibilidad en Kubernetes configurado manualmente con MicroK8s en varias máquinas virtuales EC2 (IaaS). Esta infraestructura incluye alta disponibilidad en la capa de aplicación, base de datos, y almacenamiento, con manejo de volúmenes compartidos mediante un servidor NFS, balanceo de carga y un dominio personalizado configurado con HTTPS.

### 1.1. Aspectos cumplidos o desarrollados:

- Implementación de un clúster de Kubernetes en MicroK8s sobre al menos tres instancias EC2 en AWS, permitiendo escalabilidad y alta disponibilidad para WordPress.
- Configuración de un balanceador de carga mediante un servicio de Kubernetes en los pods de WordPress, con un intento adicional de balanceo manual mediante HAProxy.
- Implementación de un sistema de almacenamiento compartido utilizando NFS, cumpliendo con los requisitos de almacenamiento externo.
- Despliegue de MySQL en el clúster como base de datos externa para WordPress, garantizando el acceso a datos de manera distribuida.
- Configuración del dominio personalizado para WordPress, reutilizando el dominio del Reto 2: `reto2telematicakubernetes.online`.
- Configuración adecuada de estilos para WordPress, logrando que el sitio se vea visualmente completo.

### 1.2. Aspectos NO cumplidos:

- El certificado SSL no se generó correctamente, y el ingreso a la aplicación a través de HTTPS no se completó exitosamente.
- El intento de balanceo mediante HAProxy no permite la conexión externa al balanceador.

---

## 2. Información general de diseño de alto nivel, arquitectura, patrones, mejores prácticas utilizadas

- **Arquitectura distribuida:** Configuración de un clúster de alta disponibilidad con MicroK8s, distribuyendo la aplicación WordPress entre múltiples nodos de EC2 para asegurar la disponibilidad y el escalamiento.
- **Balanceador de carga mediante servicio de Kubernetes:** Configuración de un servicio en los pods de WordPress para balancear las solicitudes de los usuarios de manera equitativa. También se intentó implementar HAProxy como balanceador de carga manual, aunque no se logró acceso externo.
- **Sistema de almacenamiento compartido con NFS:** Despliegue de un servidor NFS para almacenamiento compartido entre los nodos, asegurando la persistencia de datos.
- **Intento de configuración HTTPS con dominio personalizado:** Configuración de un dominio personalizado con intención de asegurar la conexión HTTPS mediante un certificado SSL (aunque este último no se generó correctamente).
- **Escalabilidad y monitoreo:** Configuración del clúster para permitir la adición de nodos de manera manual para satisfacer picos de demanda.

---

## 3. Descripción del ambiente de desarrollo y técnico

- **Lenguaje de programación y configuración:**  
  - YAML (para los manifiestos de Kubernetes y configuración de servicios).
  - Bash y AWS CLI (para la gestión y despliegue de infraestructura en AWS).

- **Herramientas y servicios en la nube:**  
  - **AWS EC2 (Elastic Compute Cloud):** Provisión de instancias para el despliegue del clúster Kubernetes.
  - **MicroK8s:** Distribución de Kubernetes utilizada para desplegar el clúster en IaaS (EC2).
  - **NFS (Network File System):** Servidor de almacenamiento utilizado para gestionar el almacenamiento compartido.
  - **Balanceador de carga:** Configurado para distribuir tráfico entre los pods de WordPress a través del servicio de Kubernetes.

- **Componentes en Kubernetes:**  
  - **NGINX Ingress Controller:** Gestiona el acceso externo al clúster a través de HTTP.
  - **MySQL (Base de datos):** Base de datos desplegada en el clúster de Kubernetes para gestionar los datos de WordPress.
  - **Persistent Volumes (PV) y Persistent Volume Claims (PVC):** Definidos para gestionar el almacenamiento de archivos de WordPress y la base de datos en el sistema de archivos NFS.

---

## 4. Configuración y despliegue en AWS EC2 con MicroK8s

### 1. Configuración del entorno en AWS

1. **Crear y configurar las instancias EC2:**
   - Crear al menos tres instancias EC2 en AWS para el clúster de Kubernetes.
   - Configurar el acceso SSH para cada una.

2. **Instalar MicroK8s en cada instancia:**
   - Actualizar e instalar MicroK8s en las instancias:
     ```bash
     sudo apt update && sudo apt upgrade -y
     sudo snap install microk8s --classic
     sudo usermod -a -G microk8s ubuntu
     sudo chown -f -R ubuntu ~/.kube
     ```
   - Habilitar los servicios necesarios en el nodo principal:
     ```bash
     sudo microk8s enable dns storage ingress metrics-server
     sudo microk8s status --wait-ready
     sudo snap refresh microk8s
     ```

3. **Agregar nodos al clúster:**
   - En el nodo principal, ejecutar:
     ```bash
     sudo microk8s add-node
     ```
   - En los nodos adicionales, ejecutar el comando proporcionado por el nodo principal para unirse al clúster.

4. **Habilitar alta disponibilidad:**
   - Ejecutar en el nodo principal para permitir la alta disponibilidad:
     ```bash
     sudo microk8s enable ha-cluster
     ```

### 2. Configuración del almacenamiento compartido con NFS

1. **Instalar NFS en el nodo principal:**
   ```bash
   sudo apt update
   sudo apt install -y rpcbind nfs-kernel-server
   sudo mkdir -p /srv/nfs/kubedata
   sudo chown nobody:nogroup /srv/nfs/kubedata
   ```
2. **Configurar exportaciones para compartir el directorio:**
   ```bash
   echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -a
   sudo systemctl restart nfs-kernel-server
   ```

3. **Montar el NFS en los otros nodos:**  
   - En cada nodo adicional:
     ```bash
     sudo apt install -y nfs-common
     sudo mkdir -p /mnt/nfs-test
     sudo mount -v -o vers=4,loud <IP_NODO_PRINCIPAL>:/srv/nfs/kubedata /mnt/nfs-test
     ```

### 3. Despliegue de MySQL con Helm en el clúster MicroK8s

1. **Habilitar Helm:**
   ```bash
   sudo microk8s enable helm3
   ```
2. **Instalar MySQL (originalmente intentamos con MariaDB pero surgieron problemas):**
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install mydb bitnami/mysql --set architecture=replication
   ```

### 4. Configuración del dominio y del ingreso HTTPS

1. **Configurar el dominio con GoDaddy:**
   - Reutilizar el dominio `reto2telematicakubernetes.online` del Reto 2, configurado en GoDaddy.

2. **Intentar generar un certificado SSL con Cert-Manager (intentado pero no completado):**
   ```bash
   microk8s kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml
   ```

---

## 5. Descripción de archivos de configuración del ambiente (ubicados en `wordpress-deployment`)

1. `000-default.conf` - Configuración de Apache para WordPress.
2. `apache2.conf` - Archivo de configuración para Apache2.
3. `cluster-issuer.yaml` - Emisor de certificados Let's Encrypt para Cert-Manager.
4. `drupal-nfs-pv-pvc.yaml` - Configuración de PV y PVC para NFS en Drupal (no utilizado finalmente).
5. `ingress.yaml` - Configuración de Ingress para WordPress.
6. `mysql-service.yaml` - Servicio de MySQL.
7. `wordpress-service.yaml` - Servicio de WordPress.
8. `wordpress.yaml` - Despliegue de WordPress.

---

## 6. Fotos de la ejecución del sistema:

[Incluye aquí las fotos de ejecución del sistema, al igual que en el README del Reto 2]

---

## 7. Intentos fallidos y problemas encontrados

- **Certificado SSL:** No se generó correctamente el certificado para HTTPS, por lo que la conexión segura no fue posible.
- **Problema con Ingress:** El Ingress no se configuró correctamente, impidiendo el acceso adecuado a la aplicación a través del dominio.
- **HAProxy como

 balanceador:** Intentamos configurar HAProxy como balanceador manual, pero no se logró acceso externo y el balanceo no fue efectivo.
- **Migración de Moodle a Drupal y luego a WordPress:** Intentamos configurar Moodle como en el Reto 2, pero surgieron problemas, por lo que pasamos a Drupal; sin embargo, la imagen estaba incompleta, así que finalmente optamos por WordPress.
- **Base de datos:** Inicialmente configuramos MariaDB, pero nos encontramos con problemas y decidimos cambiar a MySQL.

---

## 8. Video:

Haremos el video y te lo mandaremos por el correo institucional

---

## 9. Referencias

- **MicroK8s Documentation:** [https://microk8s.io/](https://microk8s.io/)
- **AWS EC2 Documentation:** [https://docs.aws.amazon.com/ec2/](https://docs.aws.amazon.com/ec2/)
- **NFS Setup Guide:** [https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04-es](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04-es)
- **WordPress Docker Documentation:** [https://hub.docker.com/r/bitnami/wordpress/](https://hub.docker.com/r/bitnami/wordpress/)
- **Persistent Volumes in Kubernetes:** [https://kubernetes.io/docs/concepts/storage/persistent-volumes/](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- **GoDaddy API for DNS Management:** [https://www.godaddy.com/help/managing-dns-680](https://www.godaddy.com/help/managing-dns-680)
- **Deploying MicroK8s on AWS:** [https://www.cloudthat.com/resources/blog/deploying-microk8s-on-aws-a-step-by-step-guide](https://www.cloudthat.com/resources/blog/deploying-microk8s-on-aws-a-step-by-step-guide)