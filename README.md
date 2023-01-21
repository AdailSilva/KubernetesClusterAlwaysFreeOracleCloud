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
