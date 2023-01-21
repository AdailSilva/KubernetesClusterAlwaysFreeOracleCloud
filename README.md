# Projeto de um Cluster Kubernetes gratuíto e vitalício

Todas as contas do [Oracle Cloud Infrastructure][oci] (gratuitas ou pagas) possuem um conjunto de recursos gratuitos durante a vida útil da conta. Esses recursos exibem o rótulo `Always Free` no console (para formas de computação Ampere A1, consulte: `Compute` [neste link][compute]).

Esta é uma configuração do Terraform para implantar um cluster Kubernetes em [Oracle Cloud Infrastructure][oci]. Ele cria algumas máquinas virtuais e usa o [kubeadm] para instalar um plano de controle do Kubernetes no primeiro máquina e junte-se às outras máquinas como nós de trabalho.

Por padrão, ele implanta um cluster de 4 nós usando máquinas ARM. Cada máquina tem 1 OCPU e 6 GB de RAM, o que significa que o cluster cabe dentro do Oracle's [free tier][freetier].

**Não se destina a executar cargas de trabalho de produção,** mas é ótimo se você quiser aprender Kubernetes com um cluster "real" (ou seja, um cluster com vários nós) sem gastar muito, *e* se você deseja desenvolver ou testar aplicativos no ARM.

## Começando

1. Crie uma conta Oracle Cloud Infrastructure (basta seguir [este link][create account]).
2. Ter instalado ou [instalar o kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl).
3. Ter instalado ou [instalar o terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/oci-get-started).
4. Ter instalado ou [instalar OCI CLI](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm).
5. Configure as [credenciais OCI](https://learn.hashicorp.com/tutorials/terraform/oci-build?in=terraform/oci-get-started).
    Se você obtiver um token de sessão (com `oci session authenticate`), certifique-se de colocar a região correta e, quando solicitado pelo nome do perfil, insira `DEFAULT` para que o Terraform encontre o token de sessão automaticamente.
6. Baixe este projeto e entre em sua pasta.
7. `terraform init`
8. `terraform apply`

É isso!

Ao final do `terraform apply`, um arquivo `kubeconfig` é gerado neste diretório. Para usar seu novo cluster, você pode fazer:

Linux
```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes
```

Windows
```powershell
$env:KUBECONFIG="$pwd\kubeconfig"
kubectl get nodes
```

O comando acima deve mostrar 4 nós, nomeados `adailsilva-node-1` a `adailsilva-node-4`.

Você também pode fazer login nas VMs. No final da saída do Terraform você deve ver um comando que pode usar para SSH na primeira VM (basta copiar e colar o comando).

## Windows

Funciona com Windows 10/Powershell 5.1.

Pode ser necessário alterar a política de execução para irrestrita.

[PowerShell ExecutionPolicy](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-5.1)

## Customização

Verifique `variables.tf` para ver os parâmetros que podem ser ajustados. Você pode alterar o número de nós, o tamanho dos nós ou alternar para instâncias Intel/AMD se você Curti. Lembre-se de que, se você alternar para instâncias Intel/AMD, não obterá vantagem do nível gratuito.

## Parando o cluster

`terraform destroy`

## Detalhes da implementação

Esta configuração do Terraform:

- Gera um par de chaves OpenSSH e um token kubeadm
- Implanta 4 VMs usando o Ubuntu 20.04
- Usa cloud-init para instalar e configurar tudo
- Instala pacotes Docker e Kubernetes
- Executa `kubeadm init` na primeira VM
- Executa `kubeadm join` nas outras VMs
- Instala o plug-in Weave CNI
- Transfere o arquivo `kubeconfig` gerado por `kubeadm`
- Corrige esse arquivo para usar o endereço IP público da máquina

## Ressalvas

Isso não instala o [OCI cloud controller manager][ccm], o que significa que você não pode criar serviços com `type: LoadBalancer`; ou melhor, se você criar tais serviços, seu `EXTERNAL-IP` permanecerá `<pending>`.

Para expor serviços, use `NodePort`.

Da mesma forma, não há controlador de entrada nem classe de armazenamento.

Eles podem ser adicionados em uma iteração posterior deste projeto. Enquanto isso, se você quiser instalá-lo manualmente, você pode verificar o [OCI cloud controller manager github repository][ccm].

## Observações

O Oracle Cloud também possui um serviço Kubernetes gerenciado chamado [Container Engine for Kubernetes (ou OKE)][oke]. Esse serviço não possui as ressalvas mencionadas acima, porém não faz parte da camada gratuita.

## Possíveis erros e como resolvê-los

### Problema de autenticação

Se você configurou a autenticação OCI usando um token de sessão (com `oci session authenticate`), observe que este token é válido 1 hora por padrão. Se você autenticar, espere mais de 1 hora, então tente `terraform apply`, você obterá erros de autenticação.

#### Sintoma

A seguinte mensagem:

```
 Error: 401-NotAuthenticated
│ Service: Identity Compartment
│ Error Message: The required information to complete authentication was not provided or was incorrect.
│ OPC request ID: [...]
│ Suggestion: Please retry or contact support for help with service: Identity Compartment
```

#### Solução

Autenticar ou autenticar novamente, por exemplo, com `oci session authenticate`.

Se for solicitado o nome do perfil, certifique-se de inserir `DEFAULT` para que o Terraform use automaticamente o token de sessão.

Se você já usou `oci session authenticate`, você deve ser capaz de atualizar a sessão com `oci session refresh --profile DEFAULT`.

### Problema de capacidade

#### Sintoma

Se você receber uma mensagem como a seguinte:
```
Error: 500-InternalError
│ ...
│ Service: Core Instance
│ Error Message: Out of host capacity.
```

Isso significa que não há servidores suficientes disponíveis no momento no OCI para criar o cluster.

#### Solução

Uma solução é mudar para um *availability domain* diferente. Isso pode ser feito alterando a variável de entrada `availability_domain`.

Nota 1: Algumas regiões possuem apenas um domínio de disponibilidade. Naquilo caso, você não pode alterar o 'availability domain'.

Observação 2: Contas OCI (especialmente contas gratuitas) estão vinculadas a um região única, portanto, se você tiver esse problema e não puder alterar o domínio de disponibilidade, você pode [create another account] [create account].

### Usando a região errada

#### Sintoma

Ao fazer `terraform apply`, você recebe esta mensagem:

```
oci_identity_compartment._: Creating...
╷
│ Error: 404-NotAuthorizedOrNotFound
│ Service: Identity Compartment
│ Error Message: Authorization failed or requested resource not found
│ OPC request ID: [...]
│ Suggestion: Either the resource has been deleted or service Identity Compartment need policy to access this resource. Policy reference: https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/policyreference.htm
│
│
│   with oci_identity_compartment._,
│   on main.tf line 1, in resource "oci_identity_compartment" "_":
│    1: resource "oci_identity_compartment" "_" {
│
╵
```

#### Solução

Edite `~/.oci/config` e altere a linha `region=` para colocar a região correta.

Para saber qual é a região correta, você pode tentar entrar no https://cloud.oracle.com/ com sua conta; depois de fazer login, você deve ser redirecionado para uma URL que se parece com https://cloud.oracle.com/?region=sa-saopaulo-1 e nesse exemplo, a região é `sa-saopaulo-1`.

### Solução de problemas de criação de cluster

Depois que as VMs forem criadas, você poderá fazer login nas VMs com o usuário `ubuntu` e a chave SSH contida no arquivo `id_rsa` que foi criado pela Terraform.

Em seguida, você pode verificar o arquivo de saída do cloud init, por exemplo como isso:
```
tail -n 100 -f /var/log/cloud-init-output.log
```


[ccm]: https://github.com/oracle/oci-cloud-controller-manager
[compute]: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm#compute
[create account]: https://bit.ly/free-oci-dat-k8s-on-arm
[freetier]: https://www.oracle.com/cloud/free/
[kubeadm]: https://kubernetes.io/docs/reference/setup-tools/kubeadm/
[oci]: https://www.oracle.com/cloud/compute/
[oke]: https://www.oracle.com/cloud-native/container-engine-kubernetes/





#### Comandos úteis
# Install Kubernetes (kubernetes tools)

## Installing kubeadm

[Link]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

    Update the apt package index and install packages needed to use the Kubernetes apt repository:

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl

    Download the Google Cloud public signing key:

    sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

    Add the Kubernetes apt repository:

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

Note: In releases older than Debian 12 and Ubuntu 22.04, /etc/apt/keyrings does not exist by default. You can create this directory if you need to, making it world-readable but writeable only by admins.





# Install Terraform

[Link]: https://developer.hashicorp.com/terraform/tutorials/oci-get-started/install-cli?in=terraform%2Foci-get-started


Ensure that your system is up to date, and you have the gnupg, software-properties-common, and curl packages installed. You will use these packages to verify HashiCorp's GPG signature, and install HashiCorp's Debian package repository.

    sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

Install the HashiCorp GPG key.

    wget -O- https://apt.releases.hashicorp.com/gpg | \
    gpg --dearmor | \
    sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

Verify the key's fingerprint.

    gpg --no-default-keyring \
    --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
    --fingerprint

The gpg command will report the key fingerprint:

/usr/share/keyrings/hashicorp-archive-keyring.gpg
-------------------------------------------------
    pub  rsa4096 2020-05-07 [SC]
    E8A0 32E0 94D8 EB4E A189  D270 DA41 8C88 A321 9F7B
    uid  [ unknown] HashiCorp Security (HashiCorp Package Signing) <security+packaging@hashicorp.com>
    sub  rsa4096 2020-05-07 [E]

The fingerprint must match E8A0 32E0 94D8 EB4E A189 D270 DA41 8C88 A321 9F7B. You can also verify the key on Security at HashiCorp under Linux Package Checksum Verification.

Add the official HashiCorp repository to your system. The lsb_release -cs command finds the distribution release codename for your current system, such as buster, groovy, or sid.

    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list

Download the package information from HashiCorp.

    sudo apt update

Install Terraform from the new repository.

    sudo apt-get install terraform

TIP: Now that you have added the HashiCorp repository, you can install Vault, Consul, Nomad and Packer with the same command.


## Verify the installation

Verify that the installation worked by opening a new terminal session and listing Terraform's available subcommands.

    terraform -help
    Usage: terraform [-version] [-help] <command> [args]

The available commands for execution are listed below.
The most common, useful commands are shown first, followed by
less common or more advanced commands. If you're just getting
started with Terraform, stick with the common commands. For the
other commands, please read the help and docs before usage.

Add any subcommand to terraform -help to learn more about what it does and available options.

* There is a need to install and configure the OCI, before executing the commands below, so that they work as expected.
    terraform plan
    terraform -help plan

Troubleshoot

If you get an error that terraform could not be found, your PATH environment variable was not set up properly. Please go back and ensure that your PATH variable contains the directory where Terraform was installed.





# Install OCI - Installing the CLI

[Link]: https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm


    bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

Note

To run a 'silent' install that accepts all default values with no prompts, use the --accept-all-defaults parameter.

Example:
    bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)" --accept-all-defaults

Run:
    source ~/.bashrc

Or run:
    sudo cp /home/adailsilva/bin/oci /usr/local/bin
    sudo cp /home/adailsilva/Apps/OracleCloud/bin/oci /usr/local/bin


Verifying the OCI CLI Installation

From a command prompt, run the following command:
    oci --version





# Configure OCI credentials.

Configure OCI credentials. If you obtain a session token (with oci session authenticate), make sure to put the correct region, and when prompted for the profile name, enter DEFAULT so that Terraform finds the session token automatically.


# Commands:

    oci session authenticate

Region:	
32: sa-saopaulo-1

Tenancy:
sa-saopaulo-1

Sign in with your Oracle Cloud Infrastructure credentials
adail101@hotmail.com
[password]



# Result (Example)
    oci session authenticate
    Enter a region by index or name(e.g.
    1: af-johannesburg-1, 2: ap-chiyoda-1, 3: ap-chuncheon-1, 4: ap-dcc-canberra-1, 5: ap-hyderabad-1,
    6: ap-ibaraki-1, 7: ap-melbourne-1, 8: ap-mumbai-1, 9: ap-osaka-1, 10: ap-seoul-1,
    11: ap-singapore-1, 12: ap-sydney-1, 13: ap-tokyo-1, 14: ca-montreal-1, 15: ca-toronto-1,
    16: eu-amsterdam-1, 17: eu-dcc-milan-1, 18: eu-frankfurt-1, 19: eu-madrid-1, 20: eu-marseille-1,
    21: eu-milan-1, 22: eu-paris-1, 23: eu-stockholm-1, 24: eu-zurich-1, 25: il-jerusalem-1,
    26: me-abudhabi-1, 27: me-dcc-muscat-1, 28: me-dubai-1, 29: me-jeddah-1, 30: mx-queretaro-1,
    31: sa-santiago-1, 32: sa-saopaulo-1, 33: sa-vinhedo-1, 34: uk-cardiff-1, 35: uk-gov-cardiff-1,
    36: uk-gov-london-1, 37: uk-london-1, 38: us-ashburn-1, 39: us-chicago-1, 40: us-gov-ashburn-1,
    41: us-gov-chicago-1, 42: us-gov-phoenix-1, 43: us-langley-1, 44: us-luke-1, 45: us-phoenix-1,
    46: us-sanjose-1): 32
    Please switch to newly opened browser window to log in!
    You can also open the following URL in a web browser window to continue:
    https://login.sa-saopaulo-1.oraclecloud.com/v1/oauth2/authorize?action=login&client_id=iaas_console&response_type=token+id_token&nonce=8674dbbc-ac89-40df-8f97-07f3b44be9ed&scope=openid&public_key=eyJrdHkiOiAiUlNBIiwgIm4iOiAidzZWNU8wdGQ3NEdYYWRaYWsxNGFLMEhjMUh5MmRIMHpDQldOZXhUdHdLWlhiQ29RbUNCV1dsUU9rRXJwUFFXVkQtUURBOE5PSlk1U0l3RW15NmFNWFRlam5fM3JIam44MVdhS1hQOXM0X25VX2xRanlBZU9HSU91bjZpeVNfU2pab1NRek82S0QtVzVraDh2VE84WmxWcXlTWHhkR0x1TC11cmRsVy1PVGw0NTBwVUI3UnlLTkczZDRTdkx3NUlUM1lUUS1sVUdFVExvR0RaWGZnLWdtRmlQNHZtM005dENsUEp3OXhyVWdWQ1JpU1pRRjd1RGltNzZVUldRZkU1ZGZCS1cxY2tkRmhtSlZLbElXSGhwQS1BX3RwSEFQN2pHc2JaMDNSalhhclF5dFhWNnBZS0lIYlJQU3plbmF6MGtMSEpVdDNhRHF6Snk3VEI5bkd0U2p3IiwgImUiOiAiQVFBQiIsICJraWQiOiAiSWdub3JlZCJ9&redirect_uri=http%3A%2F%2Flocalhost%3A8181
    Completed browser authentication process!
    
    Config written to: /home/adailsilva/.oci/config
    
    Try out your newly created session credentials with the following example command:
    
    oci iam region list --config-file /home/adailsilva/.oci/config --profile DEFAULT --auth security_token



Authorization completed! Please close this window and return to your terminal to finish the bootstrap process.





# UP command:

    terraform apply
    Enter a value: `yes`



# OUTPUT (Examples)

Apply complete! Resources: 16 added, 0 changed, 0 destroyed.

`Outputs:`
    
    ssh-with-k8s-user = <<EOT
    
    ssh -o StrictHostKeyChecking=no -i id_rsa -l k8s 129.159.52.204
    ssh -o StrictHostKeyChecking=no -i id_rsa -l k8s 164.152.51.169
    ssh -o StrictHostKeyChecking=no -i id_rsa -l k8s 144.22.161.239
    ssh -o StrictHostKeyChecking=no -i id_rsa -l k8s 168.138.145.95
    
    EOT
    ssh-with-ubuntu-user = <<EOT
    ssh -o StrictHostKeyChecking=no -l ubuntu -p 22 -i id_rsa 129.159.52.204 # node1
    ssh -o StrictHostKeyChecking=no -l ubuntu -p 22 -i id_rsa 164.152.51.169 # node2
    ssh -o StrictHostKeyChecking=no -l ubuntu -p 22 -i id_rsa 144.22.161.239 # node3
    ssh -o StrictHostKeyChecking=no -l ubuntu -p 22 -i id_rsa 168.138.145.95 # node4
    EOT





# Useful commands

    export KUBECONFIG=$PWD/kubeconfig
    
    kubectl get nodes
    
    kubectl get pods --all-namespaces
    
    kubectl get pods -n kube-system


* To leave the configuration permanently:
    cp /home/adailsilva/Apps/OracleCloudAlwaysFree/kubeconfig /home/adailsilva/.kube/config

