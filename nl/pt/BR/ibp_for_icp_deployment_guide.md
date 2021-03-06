---

copyright:
  years: 2018
lastupdated: "2018-12-07"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Guia de implementação do {{site.data.keyword.blockchainfull_notm}} Platform for ICP

***[Esta página é útil? Diga-nos.](https://www.surveygizmo.com/s3/4501493/IBM-Blockchain-Documentation)***

As redes de blockchain que são baseadas no [Hyperledger Fabric](https://hyperledger-fabric.readthedocs.io/en/release-1.2/) podem ser implementadas em uma matriz quase ilimitada de configurações para suportar tantos casos de uso. Apesar dessa maleabilidade, existe uma série de melhores práticas, especialmente em torno da **sequência** adequada para configurar e implementar componentes de rede.

Este guia de implementação mostra a sequência adequada para configurar uma rede do {{site.data.keyword.blockchainfull}} Platform no {{site.data.keyword.cloud_notm}} Private (ICP), além das melhores práticas e das coisas a ser pensadas durante a implementação. No entanto, as regras básicas ainda se aplicarão se você planejar configurar somente uma CA, um solicitador ou um peer em funcionamento. Observe que o serviço de ordenação SOLO, que implementa apenas um único nó do solicitador, não é destinado para ambientes de produção. Qualquer rede ou canal que execute o SOLO não poderá ser considerado um ambiente de "produção". No entanto, é possível implementar o peer e a CA em um ambiente de produção, especialmente quando eles estão altamente disponíveis.

**IMPORTANTE**: o processo para implementar o IBP no ICP é difícil e presume um alto grau de experiência no Fabric. Se você for novo no Fabric, no IBP ou no ICP e seu objetivo for configurar um ambiente de desenvolvimento ou prova de conceito, considere consultar o [Starter Plan](/docs/services/blockchain/starter_plan.html). Observe também que nem toda configuração de implementação potencial é suportada. Por causa da dificuldade, a suposição é que um especialista no Fabric irá passar pelo processo de implementação e de conexão dos componentes antes de passar os deveres administrativos para outras partes. Esses especialistas são o público-alvo deste guia e da documentação que temos para os componentes do gráfico Helm em geral.

## Decidindo sobre a configuração de rede

A estrutura de uma rede de blockchain deve ser ditada pelo caso de uso. Essas decisões fundamentais de negócios variarão em diferentes situações, mas vamos considerar algumas.

* Configurando um **ambiente de desenvolvimento**

  É para isso que o [Starter Plan](/docs/services/blockchain/starter_plan.html) foi construído, mas se você deseja gerenciar seus próprios componentes em um ambiente de desenvolvimento, precisará de uma configuração básica que consista em um solicitador e em um peer (com uma CA para cada organização). Para cada org, você pode decidir ter apenas um único solicitador e um único peer, semelhante à configuração da [rede fabcar](https://hyperledger-fabric.readthedocs.io/en/release-1.2/understand_fabcar_network.html). Da mesma forma, é possível decidir que você precisa apenas de um único usuário administrador para todos esses componentes, em vez de criar usuários separados para cada componente.

* Implementando **componentes de produção** para uma nova rede ou para uma rede existente

  Os componentes de produção e as redes de produção têm necessidades diferentes dos ambientes de desenvolvimento ou de provas de conceito. Por um lado, a alta disponibilidade torna-se uma prioridade. Um serviço de ordenação SOLO, que inclui somente um único solicitador (e, portanto, é um ponto único de falha) é, por definição, não destinado à produção.

**Nota:** este guia de implementação não passará por cada iteração e configuração de rede em potencial, mas dará diretrizes e regras comuns a serem consideradas.

## Configurando um cluster do ICP Kubernetes

Depois de decidir a estrutura de rede, configure um cluster Kubernetes no ICP para acomodar os casos de uso. Para obter mais informações, consulte [Configurando o {{site.data.keyword.cloud_notm}} Private](/docs/services/blockchain/ICP_setup.html).

## Configurando suas autoridades de certificação

Em redes de blockchain baseadas no Fabric, o primeiro componente que deve ser implementado é uma autoridade de certificação. Isso ocorre porque a configuração de um componente deve incluir pelo menos uma identidade do usuário que esteja autorizada a operá-la antes de o componente ser implementado.

Às vezes, isso pode ser difícil de entender conceitualmente. Um dispositivo eletrônico como um laptop, por exemplo, pode frequentemente ser acessado por qualquer um quando novo. Supõe-se que a primeira pessoa que liga o dispositivo seja o "proprietário" e que, portanto, tenha permissão para registrar e estabelecer senhas que todos os usuários subsequentes deverão conhecer. No entanto, não é assim que os componentes nas redes de blockchain baseadas no Fabric são configurados.
Os peers e solicitadores devem ter informações administrativas incorporadas a eles quando implementados pela primeira vez.

** É recomendável implementar uma CA por organização.** Por exemplo, se você implementar três peers que estejam associados a uma organização e um solicitador que esteja associado à outra organização, será necessário implementar duas autoridades de certificação. Uma CA é para a organização do peer e a outra é para a organização do solicitador.

Para evitar uma regressão infinita de autoridades de certificação (em que cada CA deve ser vinculada à outra CA, continuamente para sempre), as autoridades de certificação têm um par de nome de usuário e senha padrão associado a elas. Você especifica esse nome de usuário e senha quando a CA é implementada. Essa CA é a "CA raiz", a raiz de confiança para seu componente. Em redes de produção, é recomendável implementar pelo menos uma outra CA que obtenha seu certificado a partir dessa CA raiz e usar essa CA, que é conhecida como uma "CA intermediária", para emitir certificados para usuários e componentes. No entanto, nesta liberação, o uso de autoridades de certificação intermediárias não é suportado.

Toda organização deve ter uma autoridade de certificação  para inscrição e uma autoridade de certificação do TLS. Ao implementar uma autoridade de certificação no ICP, uma autoridade de certificação do TLS também é implementada no mesmo contêiner, por padrão. Essa autoridade de certificação do TLS gera e gerencia seus certificados TLS. A autoridade de certificação em uma rede de plano Starter ou Enterprise não contém uma autoridade de certificação do TLS, mas contém uma que é usada para inscrição. Portanto, se você planeja conectar seu peer a uma rede de plano Starter ou Enterprise, antes de implementar seu peer, também deverá [implementar uma CA no ICP](/docs/services/blockchain/howto/CA_deploy_icp.html) que servirá como a CA TLS para esse peer. Consulte também os [pré-requisitos para conectar um peer de ICP ao IBP](/docs/services/blockchain/howto/peer_deploy_ibp.html#prerequisites-peer-ibp). Observe que a CA TLS é usada apenas para emitir certs e pode ser encerrada quando essa atividade é concluída.

Para obter mais informações sobre TLS, consulte [ Protegendo a comunicação com Segurança da Camada de Transporte (TLS) ![Ícone de link externo](images/external_link.svg "Ícone de link externo")](https://hyperledger-fabric.readthedocs.io/en/release-1.3/enable_tls.html "Protegendo a comunicação com Segurança da Camada de Transporte (TLS)") na Documentação do Hyperledger Fabric.

### Preparando os MSPs para o solicitador e os peers

Como o processo do IBP for ICP é tão elaborado, é recomendável usar uma única identidade do usuário administrativo como o administrador para todos os nós do componente de rede durante a configuração inicial. Isso deve reduzir os erros de implementação e de conexão, assegurando que um usuário possa configurar a configuração e as conexões entre vários componentes para assegurar que estejam funcionando corretamente. No entanto, é extremamente importante que cada componente tenha certificados diferentes. Às vezes, a distinção aqui pode ser fácil de perder. A entidade que assina uma proposta de transação não é o administrador do peer, mas o **próprio peer**. Portanto, o peer deve ser cadastrado e ter um certificado que ele anexa ao que quer que ele faça, assim como uma chave privada que ele pode usar para gerar determinados tipos de assinaturas. Para obter mais informações sobre identidades e permissões em uma rede de blockchain baseada no Fabric, consulte [Identidade ![Ícone de link externo](images/external_link.svg "Ícone de link externo")](https://hyperledger-fabric.readthedocs.io/en/release-1.3/identity/identity.html "Identidade") e [Associação ![](images/external_link.svg  "Ícone de link externo")](https://hyperledger-fabric.readthedocs.io/en/release-1.3/membership/membership.html "Associação") na Documentação do Fabric.

O cliente Fabric CA, que deve ser instalado seguindo as instruções na [documentação sobre a operação de uma CA](/docs/services/blockchain/howto/CA_operate.html#fabric-ca-client), é usado para registrar e inscrever identidades. Essa identidade é o administrador dos componentes de rede que você implementará e permite que você mude as configurações e conecte os componentes. Se desejar transferir a administração desses componentes para outros usuários posteriormente, será possível registrar e inscrever novos administradores para esses componentes e remover a si mesmo como um administrador, se necessário.

Quando o solicitador ou o peer for implementado, um contêiner `init` que está associado ao solicitador ou peer usará um objeto secreto Kubernetes para criar o MSP para o componente. Para aprender como criar um objeto secreto, consulte [Operando uma CA](/docs/services/blockchain/howto/CA_operate.html). Como dissemos acima, lembre-se de que você precisa configurar uma CA e repetir esse fluxo para cada organização.

## Implementando solicitadores e peers

Depois que um segredo de Kubernetes é criado, você está pronto para implementar um componente. Se seu plano for estabelecer canais, é recomendável implementar o solicitador antes de quaisquer peers. Certifique-se de usar nomes de organização diferentes para todos os seus componentes.

- **[Implemente o solicitador](/docs/services/blockchain/howto/orderer_deploy_icp.html)**. Observe que, no momento, apenas o serviço de ordenação SOLO é suportado. Você possui opções no processo de implementação relacionadas ao cálculo.

- **[Implemente o peer para ser conectado a um solicitador do ICP](/docs/services/blockchain/howto/peer_deploy_icp.html)**. Inúmeras opções para implementar seu peer, incluindo o tipo de banco de dados, podem ser especificadas no gráfico Helm. Assegure-se de que o ID do MSP da organização do peer seja diferente do ID do MSP da organização do solicitador.

- **[Implemente o peer para ser conectado a uma rede IBP](/docs/services/blockchain/howto/peer_deploy_ibp.html)**. O processo para implementar um peer e conectá-lo a uma rede [Starter Plan](/docs/services/blockchain/starter_plan.html) ou [Enterprise Plan](/docs/services/blockchain/enterprise_plan.html) no {{site.data.keyword.cloud_notm}} é diferente porque ele usa Perfis de Conexão e a IU do Monitor de Rede. Observe que a organização à qual o peer pertence já deve estar associada a um canal na rede. Novamente, certifique-se de que o ID do MSP da organização do peer seja diferente do ID de MSP da organização do solicitador.

## Aumentando sua rede

Se o seu objetivo for configurar um ambiente de desenvolvimento ou uma prova de conceito, será necessário incluir organizações de peer no canal do sistema do solicitador que é criado durante a implementação do solicitador. Esse é um processo com várias etapas que utiliza cada componente e está documentado nos tópicos operacionais relevantes.

1. Antes de poder incluir uma organização, é necessário criá-la e configurar seu ID de MSP. Veja [Operando uma CA](/docs/services/blockchain/howto/CA_operate.html#deploy-orderer-peer).

2. Depois que a organização tiver sido criada, ela poderá ser incluída no canal do sistema do solicitador. Para obter mais informações, consulte [Operando um solicitador](/docs/services/blockchain/howto/orderer_operate.html#add-organizations-to-consortium).

3. Quando uma organização é listada no canal do sistema do solicitador, ela é um membro do "consórcio" e está ativada para criar um canal de aplicativo, os tipos de canais nos quais as transações ocorrem. Para obter mais informações sobre como criar um canal, consulte [Operando um peer](/docs/services/blockchain/howto/peer_operate_icp.html#peer-icp-channeltx).
