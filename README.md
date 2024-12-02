# Configuración de Entorno con Vagrant, Ansible, Nginx, Prometheus y Grafana con IPs Estáticas y Claves SSH

## Introducción
En esta guía, se detallan los pasos para configurar dos máquinas virtuales Linux utilizando Vagrant y Ansible, con IPs estáticas y autenticación SSH mediante claves públicas. Las máquinas tendrán configuraciones específicas:

- **VM1:** Servidor web con Nginx.
- **VM2:** Servidores de monitoreo con Prometheus y Grafana.

Antes de empezar se debe generar una clave SSH en tu máquina local. Si no tienes una, puedes generarla siguiendo estos pasos:

Abre una terminal en tu máquina local.

```bash 
ssh-keygen -t rsa -b 4096
```

Presiona Enter para aceptar la ubicación predeterminada del archivo de clave (normalmente ~/.ssh/id_rsa) y, si lo deseas, proporciona una frase de contraseña para proteger la clave. Una vez generada la clave, puedes encontrarla en ~/.ssh/id_rsa.pub.


## 1. Configuración del Vagrantfile
El Vagrantfile es responsable de definir las máquinas virtuales, su configuración de red y provisión inicial.

### 1.1. Crear y configurar el Vagrantfile
Abre PowerShell crear y entra al directorio donde quieres configurar vagrant:

```bash
mkdir Entrega_2
cd Entrega_2
```

Inicializa vagrant en el directorio:
```bash
vagrant init
```

Modifica tu archivo Vagrantfile con el siguiente contenido:

```ruby

Vagrant.configure("2") do |config|
  # Configuración para la primera máquina virtual (VM1)
  config.vm.define "vm1" do |vm1|
    vm1.vm.box = "ubuntu/bionic64"
    vm1.vm.network "private_network", type: "static", ip: "192.168.56.101"
    vm1.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 2
    end

    # Agregar la clave SSH pública en VM1
    vm1.vm.provision "shell" do |s|
      ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
      s.inline = <<-SHELL
        mkdir -p /home/vagrant/.ssh
        echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
        chmod 700 /home/vagrant/.ssh
        chmod 600 /home/vagrant/.ssh/authorized_keys

        mkdir -p /root/.ssh
        echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
        chmod 700 /root/.ssh
        chmod 600 /root/.ssh/authorized_keys
      SHELL
    end
  end

  # Configuración para la segunda máquina virtual (VM2)
  config.vm.define "vm2" do |vm2|
    vm2.vm.box = "ubuntu/bionic64"
    vm2.vm.network "private_network", type: "static", ip: "192.168.56.102"
    vm2.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 2
    end

    # Agregar la clave SSH pública en VM2
    vm2.vm.provision "shell" do |s|
      ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
      s.inline = <<-SHELL
        mkdir -p /home/vagrant/.ssh
        echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
        chmod 700 /home/vagrant/.ssh
        chmod 600 /home/vagrant/.ssh/authorized_keys

        mkdir -p /root/.ssh
        echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
        chmod 700 /root/.ssh
        chmod 600 /root/.ssh/authorized_keys
      SHELL
    end
  end
end
```

### 1.2. Inicializar las máquinas virtuales
Ejecuta los siguientes comandos:

```bash
vagrant up
```
Esto iniciará las máquinas virtuales con las configuraciones definidas en el Vagrantfile.

![Screenshot of a comment on a GitHub issue showing an image, added in the Markdown, of an Octocat smiling and raising a tentacle.](/images/vagrant_up.png)

## 2. Configuración de Ansible

### Paso 1: Conectar a la máquina virtual VM1
Ejecuta el siguiente comando:

```bash
vagrant ssh vm1
```

### Paso 2: Instalar Ansible en VM1
Dentro de la máquina virtual, instala Ansible:

```bash
sudo apt update
sudo apt install ansible -y
```

### Paso 3: Configurar el Inventario de Ansible
Antes de crear el inventario, se recomienda evitar directorios de escritura global: El directorio /vagrant es compartido entre el host y la VM con permisos amplios. Esto puede ser útil para compartir archivos, pero no es ideal para manejar claves privadas. Considera mover las claves privadas a un directorio más seguro dentro de la VM, como `/home/vagrant/.ssh/`:

Para mover la clave privada de **VM1**:

```bash
cp /vagrant/.vagrant/machines/vm1/virtualbox/private_key /home/vagrant/.ssh/private_key_vm1
chmod 600 /home/vagrant/.ssh/private_key_vm1
```

Para mover la clave privada de **VM2**:
```bash
cp /vagrant/.vagrant/machines/vm2/virtualbox/private_key /home/vagrant/.ssh/private_key_vm2
chmod 600 /home/vagrant/.ssh/private_key_vm2
```

Por ultimo, en la máquina virtual, crea un archivo hosts:

```bash
echo "[vm1]
192.168.56.101 ansible_ssh_user=vagrant ansible_ssh_private_key_file=/home/vagrant/.ssh/private_key_vm1


[vm2]
192.168.56.102 ansible_ssh_user=vagrant ansible_ssh_private_key_file=/home/vagrant/.ssh/private_key_vm2
" > /vagrant/hosts
```

## 3. Creación de Playbooks de Ansible

### 3.1. Playbook para Nginx en VM1
Crea el archivo `/vagrant/install_nginx.yml` en **VM1**:

```ruby
---
# Asegurar que Python esté instalado en todas las máquinas
- name: Ensure Python is installed
  hosts: all
  gather_facts: no  # Desactiva la recopilación de hechos
  become: yes
  tasks:
    - name: Install Python
      raw: sudo apt update && sudo apt install -y python3
    - name: Create symlink for Python
      raw: sudo ln -sf /usr/bin/python3 /usr/bin/python


- name: Instalar Nginx en VM1
  hosts: vm1
  become: yes
  tasks:
    - name: Actualizar caché de apt
      apt:
        update_cache: yes

    - name: Instalar Nginx
      apt:
        name: nginx
        state: present

    - name: Iniciar y habilitar Nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

### 3.2. Playbook para Prometheus y Grafana en VM2
Crea el archivo `/vagrant/install_prometheus_grafana.yml` en **VM1**:

```ruby
---
- name: Instalar Prometheus y Grafana en VM2
  hosts: vm2
  become: yes
  tasks:
    - name: Actualizar caché de apt
      apt:
        update_cache: yes

    - name: Instalar Prometheus
      apt:
        name: prometheus
        state: present

    - name: Agregar la clave del repositorio de Grafana
      apt_key:
        url: "https://packages.grafana.com/gpg.key"
        state: present

    - name: Agregar el repositorio de Grafana
      apt_repository:
        repo: "deb https://packages.grafana.com/oss/deb stable main"
        state: present
        filename: grafana

    - name: Actualizar caché después de agregar el repositorio de Grafana
      apt:
        update_cache: yes

    - name: Instalar Grafana
      apt:
        name: grafana
        state: present

    - name: Iniciar y habilitar Prometheus
      service:
        name: prometheus
        state: started
        enabled: yes

    - name: Iniciar y habilitar Grafana
      service:
        name: grafana-server
        state: started
        enabled: yes
```

# 4. Ejecutar los Playbooks
Ejecuta los playbooks desde la máquina virtual VM1:

```bash
ansible-playbook -i /vagrant/hosts /vagrant/install_nginx.yml
ansible-playbook -i /vagrant/hosts /vagrant/install_prometheus_grafana.yml
```

## 5. Verificación de Servicios
### 5.1. Nginx en VM1
Accede desde un navegador a:

```text
http://192.168.56.101
```

### 5.2. Prometheus y Grafana en VM2
Accede desde un navegador a:

- **Prometheus:** http://192.168.56.102:9090
- **Grafana:** http://192.168.56.102:3000
(Usuario/Contraseña: admin / admin).
