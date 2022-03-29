# geoserver-cluster

Criar um cluster GeoServer com load balance e infraestrutura baseada em docker.

## Como usar?

Nesta sessão você encontra informações para: iniciar e parar o cluster, alterar configurações e sincronizar configurações.

## Configurações usando interface gráfica

Usar sempre a interface gráfica do nó master. Já está mapeada no Nginx para o nó master então usar a porta 80 já irá apontar pra instância correta.

http://150.163.134.167/geoserver/web
ou
http://150.163.134.167:8000/geoserver/web

Após realizar alterações de configuração, usar o script de sincronia, sessão: "Sincronizar as configurações"

## Inciar/parar o cluster

Comandos com usuário queimadas.

```sh
cd /home/queimadas/
# comando para iniciar
./gs-cluster-startup.sh

# comando para parar
./gs-cluster-startup.sh down
```

## Sincronizar as configurações

Script de sincronia usando o recurso "reload catalog" do GeoServer.

Script em: /usr/bin/cluster_reload_catalog

```sh
cluster_reload_catalog
```

# Como o servidor foi preparado?

As próximas sessões informam o passo a passo das configurações e instalação de softwares no servidor.

## Instalação/configuração no host:

Comandos usando root (sudo su)

### Instalar zip/unzip

apt-get install zip

### Instalar docker-ce

 > seguindo esse tutorial: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04


```sh
apt install apt-transport-https ca-certificates curl software-properties-common
 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
 
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"

apt-cache policy docker-ce

apt install docker-ce

systemctl status docker
```

Para habilitar o uso de comandos docker com os usuários não root:

```sh
usermod -aG docker queimadas
usermod -aG docker suporte
```

### Instalar docker compose

Comandos com o usuário queimadas

 > seguindo esse tutorial: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04

```sh
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/download/v2.3.4/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
sudo chown $USER /var/run/docker.sock
```

### Crianção dos arquivos base

Arquivo da stack, compatível com docker compose: geoserver-cluster.yaml
Arquivos de configuração do load balance: nginx/conf.d/

Repositório privado em (este repo): https://github.com/andre-carvalho/geoserver-cluster

Clonado em: /home/queimadas/project/geoserver-cluster

Script para startup do cluster em: /home/queimadas/gs-cluster-startup.sh
