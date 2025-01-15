# cloud-init-vm-create
Como criar uma vm utilizando um template personalizado de uma ISO através do cloud init e instalar pacotes específicos.

> Comece instalando o cloud-init em primeiro lugar:

```
> apt-get install -y cloud-init cloud-utils cloud-initramfs-growroot

```

> substituir os valores em: sudo nano /etc/cloud/cloud.cfg

```
datasource_list: [ NoCloud, ConfigDrive ]
datasource:
    ConfigDrive:
        dsmode: local
    NoCloud:
        fs_label: cidata
```

> Você precisará encontrar e alterar (ou adicionar) essas vars também para que a rede seja tratada adequadamente:
```
manage_resolv_conf: true
manage_etc_hosts: true
preserve_hostname: false
```

> Se o usuário criado na vm não for ubuntu, substitua também:
```
users:
   - default
disable_root: true
```

> Certifique-se de que esses três módulos estejam em primeiro:
```
 - set_hostname
 - update_hostname
 - update_etc_hosts
```

> Salve o arquivo de configuração.

Agora remova todos os arquivos de configuração aleatórios que o cloud-init ocasionalmente decide instalar que substituirão sua própria configuração e a tornarão inútil. Essa lista de arquivos muda sempre que as compilações do cloud-init parecem, portanto, basta procurar no diretório pai para ver o que há em sua versão específica do cloud-init:

```
rm -rf /etc/cloud/cloud.cfg.d/99-installer.cfg /etc/cloud/cloud.cfg.d/90_dpkg.cfg /etc/cloud/cloud.cfg.d/subiquity-disable-cloudinit-networking.cfg

```

Remova qualquer coisa que a VM pegou e que você não deseja em todos os seus modelos. Isso também limpa todas as execuções de cloud-init que possam ter ocorrido se você reinicializou a VM, portanto, o modelo com o qual você terminará será "atualizado". Também removemos o histórico de comandos do usuário root e do usuário padrão:

```
rm -rf /etc/ssh/ssh_host_*
cloud-init clean --logs
su - ubuntu
cat /dev/null > ~/.bash_history && history -c && exit
cat /dev/null > ~/.bash_history && history -c && exit

```

> Agora você pode desligar a VM e convertê-la em um template.

# Modelo de cloud init de vm com docker instalado

```
#cloud-config			 
users:
  - name: nomedavm
    plain_text_passwd: senha
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    groups: sudo
    shell: /bin/bash
    lock_passwd: false
# Configuração de IP estático com netplan
#write_files:
#  - path: /etc/netplan/50-cloud-init.yaml
#    content: |
#      network:
#        version: 2
#        ethernets:
#          eth0:
#            dhcp4: false
#            dhcp6: false
#            addresses:
#              - 192.168.1.201/24
#            nameservers:
#              addresses: [192.168.1.1]
#            routes:
#              - to: default
#                via: 192.168.1.1
runcmd:
  - echo "AllowUsers lab" >> /etc/ssh/sshd_config
  - sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
  - rm -f /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
  - systemctl restart sshd
  - netplan apply
  - apt install xe-guest-utilities -y
  - apt update && apt upgrade
  - apt install ca-certificates curl apt-transport-https xe-guest-utilities -y
  - install -m 0755 -d /etc/apt/keyrings
  - mkdir -p /etc/apt/keyrings
  - curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
  - chmod a+r /etc/apt/keyrings/docker.asc
  - echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
  - apt-get update
  - sleep 10
  - apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  - timedatectl show --property=Timezone --value | grep -q 'America/Sao_Paulo' || sudo timedatectl set-timezone America/Sao_Paulo
  ```
