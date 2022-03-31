# geoserver-cluster

Criar um cluster GeoServer com load balance e infraestrutura baseada em docker.

Onde há referencia <user>, considerar qualquer usuário não root existente no host para o qual se deseja usar futuramente na operação do cluster.

É pré requisito:

 > Ubuntu server (testado no 20.04);

 > Acesso sudo (root);

 > Criação de um **diretório base** que servirá de ponto de montagem dos volumes para containers;

 > Instalação do docker e docker compose;

 > Clone deste repositório;

 > Criação dos script para facilitar a operação;

Utilizamos como exemplo de **diretório base** o seguinte local: /dados/geoserver/

Caso o local seja trocado, deve-se alterar o arquivo "geoserver-cluster.yaml" pois nele são mapeados os volumes de cada container.

A instalação considera dois subdiretórios:
 - /dados/geoserver/gs_datadir
 - /dados/geoserver/gs_extensions

Deve-se aplicar as seguintes permissões de acesso aos dois subdiretórios:

```sh
# 1099 é o uid e gid do usuário interno dos container
sudo chown -R 1099:1099 /dados/geoserver/gs_datadir
sudo chown -R 1099:1099 /dados/geoserver/gs_extensions
```

Ao subir o cluster pela primeira vez, o diretório **gs_datadir** deverá conter o arquivo "controlflow.properties". O Geoserver irá criar os arquivos de configuração básicos em uma instalação limpa. Pode-se organizar a inclusão de arquivos de fonte de dados (data sources), que serão mapeados nas configurações de camadas do GeoServer, em qualquer subdiretório de **gs_datadir**, exemplo:

```sh
# lembrando de aplicar as permissões ao novo diretório e todos os arquivos copiados para ele
sudo mkdir /dados/geoserver/gs_datadir/datasource
sudo chown -R 1099:1099 /dados/geoserver/gs_datadir/datasource
```

O diretório **gs_extensions** deverá conter os arquivos dos plugins conforme descrito na sessão "Instalar GeoServer plugins"

## Arquivos base

Este repositório entrega os arquivos principais necessários para a configuração do cluster.

Arquivos disponíveis neste repositório:
 - Arquivo da stack, compatível com docker compose: geoserver-cluster.yaml
 - Arquivos de configuração do load balance: nginx/conf.d/
 - Arquivo de configuração do plugin Control flow (exemplo): gs_datadir/controlflow.properties

Clonar o projeto em (exemplo de local): /home/<user>/project/geoserver-cluster

Criar os script de operação, conforme sessões de descrição de cada um:
 - Script para startup do cluster em: /home/<user>/gs-cluster-manager.sh
 - Script para reload catalog: /usr/bin/cluster_reload_catalog

## Como usar?

Nesta sessão você encontra informações para: iniciar e parar o cluster, alterar configurações e sincronizar configurações.

## Configurações usando interface gráfica

Usar sempre a interface gráfica do nó master. Já está mapeada no Nginx para o nó master então usar a porta 80 já irá apontar pra instância correta.

http://<IP ou HOSTNAME>/geoserver/web
ou
http://<IP ou HOSTNAME>:8000/geoserver/web

Após realizar alterações de configuração, usar o script de sincronia, sessão: "Sincronizar as configurações"

## Inciar/parar o cluster

Comandos com usuário <user>

```sh
cd /home/<user>/
# comando para iniciar
./gs-cluster-manager.sh

# comando para parar
./gs-cluster-manager.sh down
```

### Script gs-cluster-manager.sh

Este script não é obrigatório, mas ajuda nas tarefas de restart do cluster e consiste apenas de dois comandos docker compose, como segue:

```sh
#!/bin/bash

# muda a linha de comandos para o diretório onde está o arquivo compose.
cd /home/<user>/project/geoserver-cluster

# identifica o parametro passado ao script
if [[ "" = "$1" ]]; then
    # sem parametro, considera startup do cluster
    docker compose -f geoserver-cluster.yaml up -d
else
    # qualquer valor para o parametro, considera desligamento do cluster
    docker compose -f geoserver-cluster.yaml down
fi
```

Criar o script com o conteúdo acima e alterar permissão para execução.

```sh
# local sugerido, /home/<user>/
sudo touch /home/<user>/gs-cluster-manager.sh
chmod +x gs-cluster-manager.sh
```

## Sincronizar as configurações

Script de sincronia usando o recurso "reload catalog" do GeoServer.

Script em(sugestão de local): /usr/bin/cluster_reload_catalog

```sh
# rodar este script após cada alteração de configuração, via interface gráfica, no nó principal.
cluster_reload_catalog
```

### Script cluster_reload_catalog

Este script deve possuir uma linha de comando por instância worker existente no cluster.
Deve ficar no host onde o cluster estiver em execução. Caso a opção seja outra máquina, deve-se garantir que cada instância receba o comando de forma individual, usando a porta exposta em cada container.

```sh
#!/bin/bash
curl -u admin:geoserver -v -XPOST "http://150.163.2.253:8001/geoserver/rest/reload"
curl -u admin:geoserver -v -XPOST "http://150.163.2.253:8002/geoserver/rest/reload"
curl -u admin:geoserver -v -XPOST "http://150.163.2.253:8003/geoserver/rest/reload"
```
 > # <admin> e <geoserver> devem ser usuário e senha que possua permisão de acesso para alterar as configurações via interface gráfica de configurações do GeoServer. As credenciais de acesso criadas ao iniciar o cluster pela primeira vez, são as identificadas acima.


Criar o script com o conteúdo acima e alterar permissão para execução.

```sh
# exemplo de criação em local que disponibiliza o script de forma global no sistema
sudo touch /usr/bin/cluster_reload_catalog
sudo chmod +x /usr/bin/cluster_reload_catalog
```

# Como o servidor foi preparado?

As próximas sessões informam o passo a passo das configurações e instalação de softwares no servidor.

## Instalação/configuração no host:

Comandos usando root (sudo su)

### Instalar zip/unzip

Utilizado apenas nos casos de necessidade em descompactar arquivos como plugins extras baixados direto no servidor, por exemplo.

```sh
apt-get install zip
```

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

Para habilitar o uso de comandos docker para os usuários não root:

```sh
# rodar um para cada usuário não root que se deseja atribuir o uso de comandos docker
usermod -aG docker <user>
```

### Instalar docker compose

Comandos com o usuário <user>

 > seguindo esse tutorial: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04

```sh
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/download/v2.3.4/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose

# necessário senha de root
sudo chown <user> /var/run/docker.sock
```

### Instalar GeoServer plugins

 > Obtidos de: http://geoserver.org/release/2.16.0/

```sh
# copiar os arquivos pra a pasta de exemplo e descompactar, um plugin por pasta.
cd /dados/geoserver/gs_extensions

# criar as pastas para os plugins
mkdir /dados/geoserver/gs_extensions/controlflow
mkdir /dados/geoserver/gs_extensions/imagepyramid

# baixar e descompactar os arquivos ZIP dos plugins para as respectivas pastas

# Control Flow
cd /dados/geoserver/gs_extensions/controlflow
wget https://sourceforge.net/projects/geoserver/files/GeoServer/2.16.0/extensions/geoserver-2.16.0-pyramid-plugin.zip
unzip geoserver-2.16.0-pyramid-plugin.zip
rm geoserver-2.16.0-pyramid-plugin.zip

# Image Pyramid
cd /dados/geoserver/gs_extensions/imagepyramid
wget https://sourceforge.net/projects/geoserver/files/GeoServer/2.16.0/extensions/geoserver-2.16.0-control-flow-plugin.zip
unzip geoserver-2.16.0-control-flow-plugin.zip
rm geoserver-2.16.0-control-flow-plugin.zip
```

Após instalar um novo plugin, é necessário reiniciar o cluster. Ver sessão "Inciar/parar o cluster".


## Images Docker

As imagens docker GeoServer utilizadas aqui, foram produzidas conforme descrito neste repositório.

https://github.com/terrabrasilis/geoserver-cluster

A imagem docker Nginx, foi produzida com base neste repositório.
 
https://github.com/terrabrasilis/terrabrasilis-nginx-manager