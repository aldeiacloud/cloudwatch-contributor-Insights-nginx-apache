# CloudWatch Contribuitor para NGINX e APACHE

#### Esta configuração permitirá coletar logs de acesso aos servidores web <b>nginx/apache</b> pelo CloudWatch, separando por ip de origem, codigo do status, agente usado para o acesso e data (podemos escolher 4 itens para compor o log dentro do Contribuitor).
<p>
Link da configuração: https://www.youtube.com/aldeiacloud/

---
<b>Passo 1:</b>
- Configurar uma politica para Instancia EC2, conseguir criar/popular o log com o Agent do CloudWatch Logs, segue politica a ser atribuída a instância:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
    ],
      "Resource": [
        "*"
    ]
  }
 ]
}
```
---
<b>Passo 2:</b>
- Instalar o Agent do CloudWatch Logs (Comando para Ubuntu Server):
```sh
sudo apt update -y
cd /tmp
curl https://s3.amazonaws.com/aws-cloudwatch/downloads/latest/awslogs-agent-setup.py -O
sudo apt install python
sudo python ./awslogs-agent-setup.py --region us-east-1
```
(A explicação da configuração com o Wizard será realizada no vídeo)

---
<b>Configuração do APACHE:</b>
- Configuração do output do log format para JSON (Para servidor APACHE):<p>
Comece editando o arquivo apache2.conf para seu site/servidor encontrado em /etc/apache2. <br>Adicione a seguinte linha de código à sua área LogFormat:
```sh
LogFormat "{ \"time\":\"%t\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\", \"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\", \"referer\":\"%{Referer}i\" }" cloudwatch

```

Exemplo de adição:
```sh
# The following directives define some format nicknames for use with
# a CustomLog directive.
#
# These deviate from the Common Log Format definitions in that they use %O
# (the actual bytes sent including headers) instead of %b (the size of the
# requested file), because the latter makes it impossible to detect partial
# requests.
#
# Note that the use of %{X-Forwarded-For}i instead of %h is not recommended.
# Use mod_remoteip instead.
#
LogFormat "%v:%p %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" vhost_combined
LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined
LogFormat "%h %l %u %t \"%r\" %>s %O" common
LogFormat "%{Referer}i -> %U" referer
LogFormat "%{User-agent}i" agent
LogFormat "{ \"time\":\"%t\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\", \"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\", \"referer\":\"%{Referer}i\" }" cloudwatch

# Include of directories ignores editors' and dpkg's backup files,
# see README.Debian for details.

# Include generic snippets of statements
IncludeOptional conf-enabled/*.conf

# Include the virtual host configurations:
IncludeOptional sites-enabled/*.conf

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
"/etc/apache2/apache2.conf" 228L, 7445C 
```

- em cada .conf dos seus sites, adicione o nome cloudwatch na frente do caminho do logs access configurado, exemplo:
```sh
vim /etc/apache2/sites-enabled/000-default.conf
```
Atualmente seu arquivo pode estar indicando um CustomLog com "combined", como adicionamos a sintaxe "cloudwatch" no arquivo apache.conf para tratar o log usando json, iremos fazer uma alteração:
```sh
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
```
Altere a seguinte linha do log "access.log", trocando de "combined" para "cloudwatch":
```sh
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log cloudwatch

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
```
- Salve e reinicie o apache2.
- Agora o servidor vai alterar o tipo de log para JSON, conforme exemplo abaixo:
```json
{ "time":"[11/Aug/2014:17:21:45 +0000]", "remoteIP":"127.0.0.1", "host":"localhost", "request":"/index.html", "query":"", "method":"GET", "status":"200", "userAgent":"ApacheBench/2.3", "referer":"-" }
```
---
<b>Configuração do NGINX:</b>
- Configuração do output do log format para JSON (Para servidor NGINX):

A configuração do Nginx é quase exatamente igual à configuração do Apache2. Para configurar o Nginx para fazer login no formato JSON, adicione as seguintes linhas à seção "# Logging Settings" do seu nginx.conf (geralmente encontrado em /etc/nginx).

```json
log_format cloudwatch escape=json
  '{'
    '"time_local":"$time_local",'
    '"remote_addr":"$remote_addr",'
    '"remote_user":"$remote_user",'
    '"request":"$request",'
    '"status": "$status",'
    '"body_bytes_sent":"$body_bytes_sent",'
    '"request_time":"$request_time",'
    '"http_referrer":"$http_referer",'
    '"http_user_agent":"$http_user_agent"'
  '}';
```
Altere a linha "access_log" para agora popular para Json:<p>
Linha atual:
```sh
access_log /var/log/nginx/access.log;
```
Alterar para:
```sh
access_log /var/log/myformat-access.log cloudwatch;
```

- Salve e reinicie o nginx.
- Agora o servidor vai alterar o tipo de log para JSON, conforme exemplo abaixo:
```json
{"time_local":"08/Jul/2023:15:26:40 +0000","remote_addr":"127.0.0.1","remote_user":"","request":"GET / HTTP/1.1","status": "200","body_bytes_sent":"10918","request_time":"0.000","http_referrer":"","http_user_agent":"curl/7.68.0"}
```
---
<p>

- ### A Configuração do Dashboard estará no vídeo deste repositório no Link de referência ao topo deste artigo.<p></b>


---
<b>Observações para as keys do Contributor:</b>
+ Você pode escolher ate 4 parametros para compor o contributor, você escolhe eles através dos parametros do log.
+ No apache se usa "remoteIP" para o ip do servidor remoto e no nginx "remote_addr"

<b>Observações para o Cloudwatch Agent:</b>
+ Não esqueça de colocar as políticas para que a instância consiga popular os Logs
+ Recomendo colocar uma retenção no grupo de logs criado
+ O nome do log que você colocar no Wizard, será o nome do LogGroup a ser criado no console AWS.
