Passo a passo: Instalar NGINX em EC2 usando Ansible

1. Criar as instâncias EC2
Você pode usar AWS Console, CLI ou Terraform.
Certifique-se de:
AMI Ubuntu/Debian (ou Amazon Linux 2, ajustando comandos)
Porta 80 liberada no Security Group
Chave SSH funcionando
IP público habilitado ou acessar via bastion

2. Instalar o Ansible na sua máquina de controle
Exemplo no Ubuntu local:
sudo apt update
sudo apt install ansible -y

Verifique:
ansible --version

3. Criar o arquivo de inventário
Crie um arquivo hosts.ini:
[nginx_servers]
ec2-1 ansible_host=3.92.100.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/my-key.pem
Para Amazon Linux:
ansible_user=ec2-user

4. Testar a conexão
ansible -i hosts.ini nginx_servers -m ping
Se der pong, a conexão está ok.

5. Criar o playbook Ansible
Crie o arquivo install_nginx.yml:
---
- name: Instalar e configurar NGINX </br>
  hosts: nginx_servers
  become: yes

  tasks:

    - name: Atualizar pacotes </br>
      apt:</br>
        update_cache: yes</br>
      when: ansible_os_family == "Debian"</br>

    - name: Instalar NGINX (Debian/Ubuntu) </br>
      apt:</br>
        name: nginx</br>
        state: present</br>
      when: ansible_os_family == "Debian"</br>

    - name: Instalar NGINX (Amazon Linux 2) </br>
      yum:</br>
        name: nginx</br>
        state: present</br>
      when: ansible_os_family == "RedHat"</br>

    - name: Iniciar e habilitar NGINX </br>
      service:</br>
        name: nginx</br>
        state: started</br>
        enabled: yes</br>
      
  handlers: </br>
    - name: restart nginx</br>
      service:</br>
        name: nginx</br>
        state: restarted </br>
      
Esse playbook funciona tanto para Ubuntu/Debian quanto para Amazon Linux 2.

6. Executar o playbook
ansible-playbook -i hosts.ini install_nginx.yml

7. Verificar
Depois da execução:
curl http://3.92.100.10

Opcional: Criar página customizada
Adicione ao playbook:</br>
    - name: Criar página HTML personalizada </br>
      copy:</br>
        dest: /var/www/html/index.html</br>
        content: "<h1>NGINX instalado com Ansible!</h1>"</br>

Algumas observações importantes:

1. O que são handlers no Ansible?
Handlers são ações que só são executadas quando algum task notifica que algo mudou.

Exemplo no playbook:
handlers:</br>
  - name: restart nginx</br>
    service:</br>
      name: nginx</br>
      state: restarted</br>

Esse handler apenas roda quando um task usa:
notify: restart nginx

Por que isso é útil?
Imagine que:
você instala NGINX
você altera uma configuração
você só quer reiniciar o serviço quando algo realmente mudou
Sem handlers, você possivelmente faria restart sempre, o que não é ideal.

Handlers → ajudam a manter o idempotency e evitar reinícios desnecessários.

2. Explicação de cada trecho do playbook
   Cabeçalho
- name: Instalar e configurar NGINX em EC2 </br>
  hosts: nginx_servers</br>
  become: yes</br>


hosts: define quais máquinas do inventário serão afetadas

become: diz que as tarefas precisam usar sudo

  Atualizar pacotes
- name: Atualizar pacotes </br>
  package:</br>
    update_cache: yes</br>

O módulo package é genérico e funciona para apt, yum, etc.

  Instalar NGINX</br>
- name: Instalar NGINX </br>
  package:</br>
    name: nginx</br>
    state: present</br>
  notify: restart nginx</br>


Se o pacote já estiver instalado → nada muda, e o handler não é chamado.
Se instalar / atualizar → o handler será acionado.

  Criar página HTML
- name: Página HTML customizada </br>
  copy:</br>
    dest: /var/www/html/index.html</br>
    content: "<h1>Deploy via Ansible!</h1>"</br>

Se o arquivo já existir e estiver igual → Ansible não faz nada.
Se mudar → handler é acionado.

  Handler
handlers:
  - name: restart nginx </br>
    service:</br>
      name: nginx</br>
      state: restarted</br>

Só executa quando notificado.

3. Como tornar mais seguro o uso da chave .pem
Existem várias maneiras de proteger ou até substituir o uso de chaves SSH tradicionais. Vamos por níveis de segurança:
Nível 1 — Segurança básica com .pem
Permissões corretas
chmod 400 ~/.ssh/minha-chave.pem

Não colocar o caminho no hosts.ini

Em vez de:
ansible_ssh_private_key_file=~/.ssh/key.pem

Use:
ansible_ssh_private_key_file={{ lookup('env','SSH_KEY') }}
E defina a variável no shell:
export SSH_KEY=~/.ssh/key.pem

Agora o inventário fica seguro e sem caminhos sensíveis.
Usar Ansible Vault para criptografar o inventário
ansible-vault encrypt hosts.ini

Agora muitas pessoas podem usar o inventário sem ver o conteúdo real.

Nível 2 — Não armazenar chave no inventário
Você pode definir no ansible.cfg:
[defaults]
private_key_file=~/.ssh/my-key.pem
E remover completamente do inventário.

Nível 3 — Usar SSH Agent (NÃO expor chave para o Ansible)
Carregue a chave no agente local:
eval $(ssh-agent)
ssh-add ~/.ssh/key.pem
E remova ansible_ssh_private_key_file do inventário.
Ansible usa a chave via agent automaticamente.

Nível 4 — NÃO usar chave .pem (recomendado)
Alternativas melhores:
A) Conectar por AWS SSM (sem SSH, sem chave, mais seguro)

Essa é hoje a forma mais segura e corporativa.

Requisitos na EC2:
Papel IAM com AmazonSSMManagedInstanceCore
SSM agent ativo

No Ansible:
ansible_connection=aws_ssm

Ou:
[nginx_servers]
i-0abc123 ansible_connection=aws_ssm

Vantagens:
Sem abrir porta 22
Sem chave SSH
Sem IP público
Alta segurança

B) SSH via IAM — EC2 Instance Connect
Permite gerar chaves temporárias automaticamente.
Você não armazena .pem.
AWS fornece um comando que cria um SSH temporário por 60 segundos.
Pode automatizar isso no Ansible usando:
ansible_ssh_common_args='-o "IdentitiesOnly=yes" -o "ProxyCommand=aws ec2-instance-connect ... "'

C) Criar chaves efêmeras via SSM
Ansible pode gerar chaves temporárias, subir para a EC2 e depois revogar.

D) Criar um bastion (jump host)
Você guarda sua privada no bastion, e Ansible faz proxy SSH:

ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p ubuntu@bastion"'

Assim, sua máquina local não precisa da chave.
