# Roteiro de Instalação de uma [VPN](https://www.kaspersky.com.br/resource-center/definitions/what-is-a-vpn "Virtual Private Network") [P2P](https://pt.wikipedia.org/wiki/Peer-to-peer "Peer-to-peer") com [OpenVPN](https://openvpn.net/community-resources/how-to/#openvpn-quickstart "OpenVPN")  

  O cenário hipotético contém 3 [_host_](https://www.techtudo.com.br/noticias/2012/02/o-que-e-um-host.ghtml), cada um deles com 2 interfaces de rede, nas quais uma receberá um endereço "externo" e outra o endereço interno (que simulará [LANs](https://pt.wikipedia.org/wiki/Rede_de_%C3%A1rea_local "Local Area Network") independentes).  
  A distribuição utiliza foi o Debian 11.  

## Ajuste da identificação do _hosts_ na rede.  

### 1. Mudar o nome da máquina  

**Editar o arquivo /etc/hostname**  

   ```#hostname [NovoNomeDoHost]; hostname > /etc/hostname```  

   Após a mudança, o _prompt_ ainda mantém o _hostname_ anterior, para fazer a atualização basta encerrar a seção e iniciar outra.  

### 2. Editar o arquivo _hosts_.  
  
  Esta edição fará a relação entre o nome e o endereço dos _hosts_.  

**Editar o arquivo /etc/hosts.**  
  
  Neste arquivo, além da identificação do _localhost_, podem ser inseridos nomes para os principais _hosts_ da rede. Esta identificação com o nome de outros _hosts_ é para "suprir" a inexistência de um [DNS](https://www.cloudflare.com/pt-br/learning/dns/what-is-dns/ "Domain Name System") e facilitar o acesso através dos nomes e não, somente, com os endereços.  

  ```#vi /etc/hosts```  

  O conteúdo inicial deste arquivo é:  
   
```  
127.0.0.1	localhost

# The following lines are desirable for IPv6 capable hosts
::1	localhost	ip6-localhost	ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```  

  No cenário criado, as interfaces externas receberão ips de uma rede 192.168.0.248/28 (que simularão uma rede com ips públicos e devem ser ajustados de acordo com o ambiente simulado, ou real) e as internas receberão 10.1.0.1/16, 10.2.0.1/16 e 10.3.0/16 para identificar cada uma das LANs, apesar de parecerem uma sequência eles não precisam ter relação alguma.  
  
  Mais detalhes serão vistos na seção de configuração das interfaces.  

```  
127.0.0.1	localhost

# The following lines are desirable for IPv6 capable hosts
::1	localhost	ip6-localhost	ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters  

# Hosts da Minha WAN
192.168.0.249	servA.wan	servA  
192.168.0.250	servB.wan	servB  
192.168.0.251	servC.wan	servC  

# Gateways das LANs
10.1.0.1	gwlanA.intra	gwlanA  
10.2.0.1	gwlanB.intra	gwlanB
10.3.0.1	gwlanC.intra	gwlanC
```  

  Este arquivo _hosts_,inicialmente, será comum para as 3 máquinas utilizadas. Ele viabiliza os testes de conectividade pelos nomes, o que reduz a possibilidade de erros que podem ocorrer tanto na digitação dos endereços ip quanto na identificação da interface desejada.  
  
## Configurações das interfaces de redes  
  
  As configurações das interfaces são feitas no arquivo [`/etc/network/interfaces`](https://linuxhint.com/debian_etc_network_interfaces/) em cada um dos _hosts_ .  
  Na criação deste ambiente de teste, foi feita a configuração do _host_ servA, a instalação dos programas necessários, a replica para o servB e o servC, e, em seguida, a edição dos endereços de rede e dos ajustes nos arquivos necessários.  
  
**Editar o arquivo /etc/hostname**  
  
  O conteúdo original do arquivo:  
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback


# DHCP on interface enp0s3

auto enp0s3
    allow-hotplug enp0s3
    iface enp0s3 inet dhcp

auto enp0s8
    allow-hotplug enp0s8
    iface enp0s8 inet dhcp
```  
  
  No servA, o arquivo ficou:  
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet static
	address 192.168.0.249
	netmask 255.255.255.248
	network 192.168.0.248
	broadcast 192.168.0.255
	gateway 192.168.0.1
	dns-nameservers 192.168.0.1

# The secundary network interface
auto enp0s8
iface enp0s8 inet static
	address 10.1.0.1
	netmask 255.255.0.0
	network 10.1.0.0
	broadcast 10.1.255.255
```  
  
**Ativação do roteamento de pacotes entre as redes do host.**  
  
  Para que haja a comunicação entre as duas [redes configuradas no _host_](https://linuxeprogramacao.blogspot.com/2013/12/compartilhando-internet-com-iptables.html) é necessário:  
  
### 1. Permitir o roteamento de pacotes.  
  + Digitar o comando:  
    `#echo 1 > /proc/sys/net/ipv4/ip_forward`  
  + Editar o arquivo `etc/sysctl.conf`, removendo o comentário da linha com o texto:  
    `net.ipv4.ip_forward=1`  
  + Carregar o novo parâmetro para o sistema, digitando:  
    `#sysctl -p`

  
### 2. Ajuste do [firewall](https://wiki.debian.org/DebianFirewall) com o netfilter utilizando o comando iptables.  
  
  Para configurar a ofscação do ip basta digitar somente:  
  `#iptables -t nat -A POSTROUTING -o enp0s3 -i enp0s8 -s 10.1.0.0/16 -j MASQUERADE`  
  
  O parâmtro `-o`, de output, deve ser seguido da interface de saída (com ip externo), o `-i`, de input, é a interface da LAN, e o `-s`, de source, é a rede da LAN seguido da máscara.  
  

## Programas essenciais  
  
  Os _hosts_ que atuam como servidores são pontos sensíveis e, por este motivo, a instalação de pacotes e de serviços deve ser realizada com cuidados. Duas práticas bastante relevantes: o registro, a documentação, dos programas e serviços instalados; e a feitura de cópias dos arquivos de configuração antes de alterar. Por padrão, o Linux mantém uma cópia do estado anterior (indicada com um `~` no final do nome do arquivo), mas talvez pretenda-se restaurar algum ponto que foi modificado, porém indisponível na cópia feita pelo sistema.  
  
  Deste modo, com foco na segurança e na eficiência, as instalações padrões do Linux para servidores não trazem programas comumente usados para o acesso e para a administração local ou remota desses _hosts_.  
  Uns desses serviços são o [ssh](https://wiki.debian.org/SSH), para acesso remoto, o [ntp](https://wiki.debian.org/NTP) e [ntpdate](https://manpages.debian.org/testing/ntpdate/ntpdate.8.en.html), essenciais para o sincronismo em ambientes de redes, o [tcpdump](https://manpages.debian.org/testing/tcpdump/tcpdump.8.en.html) e outras ferramentas para análise de tráfego e gerência do sistema.  
  
  Na sequência, serão demonstradas a instalação e a configuração dos programas citados e de outros complementares igualmente relevantas, que os quais o detalhamento extrapola o escopo deste documento.  
  

