# Guia Profissional de Diagramas de Arquitetura AWS (draw.io)

> Guia reutilizável para produzir diagramas de arquitetura AWS no nível de
> arquitetos sênior — com ícones oficiais, contêineres aninhados corretos,
> Security Groups como fronteiras, route tables, semântica de conexões e
> legenda. Baseado na convenção **AWS Architecture Icons** e no shape set
> `mxgraph.aws4` do draw.io.

---

## 1. Princípios de um bom diagrama (o que separa amador de sênior)

1. **Containment conta a história.** A hierarquia visual deve refletir a
   hierarquia real: `AWS Cloud ⊃ Region ⊃ VPC ⊃ Subnet ⊃ Security Group ⊃
   Recurso`. Nunca "empilhe" caixas soltas; use contêineres aninhados de verdade
   (parent/child), com coordenadas relativas ao pai.
2. **Security Group é fronteira, não legenda.** Desenhe o SG como uma moldura
   tracejada **em volta** do recurso que ele protege — nunca como um post-it de
   texto. Essa é a diferença mais visível entre um diagrama júnior e um sênior.
3. **Ícones oficiais, sempre.** Use o conjunto `aws4` (atual). Evite caixas
   genéricas rotuladas quando existe o ícone do serviço.
4. **A conexão só existe se carrega significado.** Cada aresta deve representar
   um fluxo (tráfego), uma associação (route table → subnet) ou uma relação
   (assume role). Detalhe secundário vai no rótulo do nó, não em mais arestas.
5. **Cor e estilo são linguagem — seja consistente e inclua legenda.** Defina um
   significado por estilo de linha/cor e explique numa legenda.
6. **Rotule com identidade real.** IDs e nomes reais (`vpc-0e59...`,
   `taskiro-alb`, `dd31vjm8yrbep.cloudfront.net`) tornam o diagrama auditável e
   útil para operação/runbook, não apenas decorativo.
7. **Mostre o que está reservado.** Subnets de HA vazias (ex.: 2ª AZ sem
   recursos) devem aparecer rotuladas como reservadas — comunica intenção de
   design e não parece "esquecido".

---

## 2. Ferramenta: draw.io via MCP (Power `drawio`)

Dois fluxos possíveis:

- **`create_diagram`** — recebe `xml` (mxGraphModel) **ou** `mermaid`. Para
  arquitetura AWS use **sempre XML** (Mermaid não controla ícones/contêineres).
- **`search_shapes`** — busca o catálogo (10k+ shapes) e devolve o `style` exato
  + `w`/`h`. **Use isto antes de desenhar** para pegar o style string atual de
  cada ícone, em vez de decorar.

### Passos de layout/roteamento (escolha UM)

| Parâmetro | Quando usar | Efeito |
|---|---|---|
| _(nenhum)_ | nós espaçados, poucas arestas | roteador básico (linhas retas, sem desvio de obstáculo) |
| `routing: "libavoid"` | **arquitetura, rede, layouts posicionados à mão** | mantém suas posições e só recalcula os fios em ângulos retos, contornando as caixas |
| `postLayout: "elk"` | fluxogramas / processos hierárquicos | **re-posiciona** os nós num layout em camadas |

> Para diagramas de arquitetura AWS a regra é: **posicione à mão + `routing:
> "libavoid"`**. Nunca use `postLayout: "elk"` aqui — ele jogaria fora o seu
> posicionamento espacial (que carrega significado).

### Regras de XML que evitam diagrama quebrado

- XML **bem-formado**, sem comentários `<!-- -->`.
- Escape em valores de atributo: `&lt; &gt; &amp; &quot;`. Quebra de linha em
  rótulo: `&#xa;` (nunca `\n`).
- **Toda aresta** precisa do filho de geometria — nunca self-closing:
  ```xml
  <mxCell id="e1" edge="1" parent="1" source="a" target="b" style="...">
    <mxGeometry relative="1" as="geometry"/>
  </mxCell>
  ```
- Arestas entre nós em **contêineres diferentes** devem ter `parent="1"` (raiz),
  senão são recortadas pelo contêiner.
- Não adicione waypoints (`<Array as="points">`) nem `exitX/entryX` — o
  `libavoid` calcula.

---

## 3. Anatomia de containment (a espinha dorsal)

Ordem de aninhamento e quem é `parent` de quem:

```
1 (camada raiz)
└─ cloud         (group AWS Cloud)
   └─ region     (group Region)
      ├─ vpc     (group VPC)
      │  ├─ igw                 (ícone, filho da VPC)
      │  ├─ pub_a (swimlane subnet pública)
      │  │  ├─ albsg (group Security Group)
      │  │  │  └─ alb  (ícone ALB)
      │  │  └─ nat   (ícone NAT — fora do SG, pois NAT não tem SG)
      │  ├─ pub_b (swimlane subnet pública, reservada HA)
      │  ├─ priv_a (swimlane subnet privada)
      │  │  └─ ec2sg (group Security Group)
      │  │     └─ ec2 (ícone EC2)
      │  ├─ priv_b (swimlane subnet privada, reservada HA)
      │  ├─ rtpub  (ícone Route Table)
      │  └─ rtpriv (ícone Route Table)
      ├─ s3 / iam / ssm  (serviços regionais fora da VPC)
      └─ legend
```

- **Coordenadas dos filhos são relativas ao pai** (origem 0,0 = canto
  superior-esquerdo do pai, abaixo da barra de título).
- Serviços globais/edge (CloudFront) e o ator (User) ficam **fora** da nuvem,
  no `parent="1"`.
- S3/IAM/SSM são regionais mas **fora da VPC** — coloque-os na `region`, ao lado
  da VPC.

---

## 4. Style strings de referência (conjunto `aws4`)

> Sempre confirme com `search_shapes` (os nomes evoluem), mas estes são os atuais.

### 4.1 Contêineres de grupo (todos `container=1;pointerEvents=0;collapsible=0`)

O esqueleto comum do grupo AWS:

```
points=[[0,0],[0.25,0],[0.5,0],[0.75,0],[1,0],[1,0.25],[1,0.5],[1,0.75],[1,1],[0.75,1],[0.5,1],[0.25,1],[0,1],[0,0.75],[0,0.5],[0,0.25]];
outlineConnect=0;gradientColor=none;html=1;whiteSpace=wrap;fontSize=12;fontStyle=0;
container=1;pointerEvents=0;collapsible=0;recursiveResize=0;
shape=mxgraph.aws4.group;grIcon=<ÍCONE>;strokeColor=<COR>;fillColor=none;
verticalAlign=top;align=left;spacingLeft=30;fontColor=<COR>;
```

| Contêiner | `grIcon` | `strokeColor` / `fontColor` | dashed |
|---|---|---|---|
| AWS Cloud | `mxgraph.aws4.group_aws_cloud_alt` | `#232F3E` | não |
| Region | `mxgraph.aws4.group_region` | `#00A4A6` / `#147EBA` | **sim** (`dashed=1`) |
| VPC | `mxgraph.aws4.group_vpc2` | `#8C4FFF` | não |
| Security Group | `mxgraph.aws4.group_security_group` | `#DD3522` | **sim** (`dashed=1`) |

Para **subnets**, uma opção robusta e legível é o `swimlane` colorido (título no
topo), em vez de adivinhar o `grIcon` de subnet:

```
# Subnet pública (verde)
swimlane;startSize=26;fillColor=#E9F5E1;strokeColor=#7AA116;fontColor=#3B6E0E;html=1;verticalAlign=top;fontSize=11;

# Subnet privada (azul)
swimlane;startSize=26;fillColor=#E6F2F8;strokeColor=#00A4A6;fontColor=#147EBA;html=1;verticalAlign=top;fontSize=11;
```

> Dica: se o rótulo da subnet tem 2 linhas (ex.: nota de HA), aumente
> `startSize` para `40` para o texto não invadir o conteúdo.

### 4.2 Ícones de recurso (`resourceIcon` — o "quadradinho" colorido)

Esqueleto do resourceIcon:

```
sketch=0;points=[[0,0,0],[0.25,0,0],...];outlineConnect=0;fontColor=#232F3E;
fillColor=<COR_CATEGORIA>;strokeColor=#ffffff;dashed=0;verticalLabelPosition=bottom;
verticalAlign=top;align=center;html=1;fontSize=11;aspect=fixed;
shape=mxgraph.aws4.resourceIcon;resIcon=mxgraph.aws4.<NOME>;
```

| Serviço | `resIcon` | `fillColor` (categoria) |
|---|---|---|
| CloudFront | `cloudfront` | `#8C4FFF` (Networking) |
| EC2 | `ec2` | `#ED7100` (Compute) |
| IAM | `identity_and_access_management` | `#DD344C` (Security) |
| Systems Manager | `systems_manager` | `#E7157B` (Mgmt) |

### 4.3 Ícones simples (sem moldura — `shape=mxgraph.aws4.<nome>`)

```
sketch=0;outlineConnect=0;fontColor=#232F3E;gradientColor=none;fillColor=<COR>;
strokeColor=none;dashed=0;verticalLabelPosition=bottom;verticalAlign=top;
align=center;html=1;fontSize=11;aspect=fixed;pointerEvents=1;shape=mxgraph.aws4.<NOME>;
```

| Serviço | `shape` | `fillColor` |
|---|---|---|
| Application Load Balancer | `application_load_balancer` | `#8C4FFF` |
| NAT Gateway | `nat_gateway` | `#8C4FFF` |
| Internet Gateway | `internet_gateway` | `#8C4FFF` |
| Route Table | `route_table` | `#8C4FFF` |
| S3 Bucket | `bucket` | `#7AA116` (Storage) |
| User | `user` | `#232F3D` |

### 4.4 Paleta de cores por categoria AWS (use para coerência)

| Categoria | Cor | Exemplos |
|---|---|---|
| Networking & Content Delivery | `#8C4FFF` | VPC, ALB, NAT, IGW, CloudFront, Route Table |
| Compute | `#ED7100` / `#D86613` | EC2, Lambda |
| Storage | `#7AA116` / `#277116` | S3, EBS |
| Security, Identity & Compliance | `#DD344C` / `#C7131F` | IAM, SG (borda `#DD3522`) |
| Management & Governance | `#E7157B` | Systems Manager, CloudWatch |
| Database | `#C925D1` | RDS, DynamoDB |
| Region (borda) | `#00A4A6` | contêiner de região |

---

## 5. Security Groups como fronteira (a melhoria-chave)

**Errado (júnior):** uma caixa de texto "SG: porta 80 do CloudFront".

**Certo (sênior):** um contêiner `group_security_group` (tracejado vermelho, com
cadeado) **envolvendo** o recurso, com o rótulo descrevendo a regra de ingress:

```xml
<mxCell id="albsg" value="taskiro-alb-sg&#xa;ingress 80 &lt;- prefix list CloudFront"
  style="...shape=mxgraph.aws4.group;grIcon=mxgraph.aws4.group_security_group;strokeColor=#DD3522;fillColor=none;dashed=1;fontColor=#DD3522;..."
  vertex="1" parent="pub_a">
  <mxGeometry x="15" y="40" width="160" height="180" as="geometry"/>
</mxCell>
<mxCell id="alb" value="ALB taskiro-alb" style="...resIcon...application_load_balancer..."
  vertex="1" parent="albsg">
  <mxGeometry x="41" y="45" width="78" height="78" as="geometry"/>
</mxCell>
```

Regras:
- O recurso vira **filho** do SG (`parent="albsg"`).
- Recursos **sem** SG (ex.: NAT Gateway, IGW) ficam **fora** da moldura — não os
  enfie num SG só por estarem na mesma subnet.
- O rótulo do SG carrega a **regra de ingress** (porta + origem). Egress padrão
  (all) normalmente é omitido para não poluir.

---

## 6. Route Tables (associação ≠ fluxo de tráfego)

Represente cada route table como ícone `route_table` posicionado perto das
subnets que ela serve, e ligue à *gateway de destino* com um estilo **distinto**
do tráfego de aplicação:

- **RT pública** → rota `0.0.0.0/0` para o **IGW**.
- **RT privada** → rota `0.0.0.0/0` para o **NAT**.
- Estilo da linha de rota: pontilhado fino cinza, para não competir com o fluxo
  de requisição:
  ```
  edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;dashed=1;dashPattern=1 2;
  strokeColor=#879196;strokeWidth=1;fontSize=10;endArrow=open;
  ```
- No rótulo do ícone, diga a associação: "RT privada — assoc. subnets privadas".

> Distinguir **configuração de roteamento** (route table → gateway) de **fluxo de
> tráfego** (ex.: EC2 → NAT → IGW) evita que o leitor confunda as duas coisas.

---

## 7. Semântica de conexões + legenda

Convencione um significado por estilo e **inclua a legenda**:

| Significado | Estilo sugerido |
|---|---|
| Tráfego de requisição (HTTPS/HTTP) | linha cheia, cor da categoria (`#8C4FFF`) ou preto, `strokeWidth=2` |
| Egress p/ internet (NAT/IGW) | tracejado verde `#7AA116`, `strokeWidth=2` |
| Rota default (route table) | pontilhado cinza `#879196`, `dashPattern=1 2` |
| Ações auxiliares (boot/IAM/SSM) | tracejado fino na cor do alvo, `strokeWidth=1` |

Caixa de legenda (texto com quadrados coloridos via `<font color>`):

```xml
<mxCell id="legend"
  value="&lt;b&gt;Legenda&lt;/b&gt;&#xa;&lt;font color='#8C4FFF'&gt;&#9632;&lt;/font&gt; Trafego de requisicao&#xa;&lt;font color='#7AA116'&gt;&#9632;&lt;/font&gt; Egress p/ internet&#xa;&lt;font color='#879196'&gt;&#9632;&lt;/font&gt; Rota default (route table)&#xa;&lt;font color='#C7131F'&gt;&#9632;&lt;/font&gt; Acoes auxiliares"
  style="text;html=1;fontSize=11;align=left;verticalAlign=top;strokeColor=#879196;fillColor=#F7F7F7;rounded=1;spacingLeft=8;spacingTop=6;"
  vertex="1" parent="region">
  <mxGeometry x="1010" y="560" width="250" height="150" as="geometry"/>
</mxCell>
```

Rótulos de aresta: curtos (1–3 palavras ou a regra: `HTTP:80`, `0.0.0.0/0`,
`assume role`). Empurre o detalhe longo para o rótulo do nó.

---

## 8. Layout e espaçamento (lições práticas)

- **Dê largura aos contêineres antes de povoar.** Falta de espaço é a causa nº 1
  de sobreposição. Para 2 colunas de subnets + coluna de serviços + legenda,
  ~1300–1350px de largura na nuvem funcionam bem.
- **Deixe uma faixa central** entre as colunas de subnet (≥100px) para alocar as
  route tables e deixar os fios respirarem.
- **`startSize`** do swimlane = altura da barra de título: 26 para 1 linha, ~40
  para 2 linhas de rótulo.
- **Ícones**: resourceIcon ~78×78; ícones simples 60–78; route table ~60×58.
- **Serviços regionais** (S3/IAM/SSM) numa coluna à direita da VPC, empilhados
  com ~170px de passo vertical.
- Rode com **`routing:"libavoid"`** para os fios contornarem as caixas.

---

## 9. Armadilhas que parecem erro mas não são

- **"overlapping_cells" no validador** entre `cloud`/`region`/`vpc`/subnets é o
  **aninhamento intencional** (pai contém filho). Não é defeito.
- **"orphan_edge"/"disconnected_vertex"** em alguns validadores aparece para
  contêineres/arestas válidas — confira visualmente antes de "consertar".
- **Renderização inline no chat** depende do host suportar MCP Apps. Se não
  aparecer imagem, **salve como `.drawio`** e abra na extensão do VS Code
  (`hediet.vscode-drawio`) ou em https://app.diagrams.net. Esse é o caminho
  confiável de visualização.
- **Edição pós-criação por id às vezes falha** em algumas ferramentas MCP; é
  mais robusto **regenerar o XML completo** do que aplicar muitos micro-edits.

---

## 10. Esqueleto reutilizável (copie e adapte)

```xml
<mxGraphModel adaptiveColors="auto" grid="1" gridSize="10">
  <root>
    <mxCell id="0"/>
    <mxCell id="1" parent="0"/>

    <!-- Ator + edge service (fora da nuvem) -->
    <mxCell id="user" value="Usuario" style="sketch=0;...shape=mxgraph.aws4.user;fillColor=#232F3D;" vertex="1" parent="1"><mxGeometry x="70" y="60" width="70" height="70" as="geometry"/></mxCell>
    <mxCell id="cf" value="CloudFront" style="sketch=0;...resIcon=mxgraph.aws4.cloudfront;fillColor=#8C4FFF;" vertex="1" parent="1"><mxGeometry x="70" y="200" width="78" height="78" as="geometry"/></mxCell>

    <!-- Cloud > Region > VPC -->
    <mxCell id="cloud" value="AWS Cloud - Conta XX  " style="...grIcon=mxgraph.aws4.group_aws_cloud_alt;strokeColor=#232F3E;..." vertex="1" parent="1"><mxGeometry x="320" y="60" width="1340" height="900" as="geometry"/></mxCell>
    <mxCell id="region" value="Region: us-east-1" style="...grIcon=mxgraph.aws4.group_region;strokeColor=#00A4A6;dashed=1;..." vertex="1" parent="cloud"><mxGeometry x="30" y="50" width="1280" height="810" as="geometry"/></mxCell>
    <mxCell id="vpc" value="VPC - 10.0.0.0/16" style="...grIcon=mxgraph.aws4.group_vpc2;strokeColor=#8C4FFF;..." vertex="1" parent="region"><mxGeometry x="30" y="50" width="940" height="720" as="geometry"/></mxCell>

    <!-- Subnets, SGs, recursos, route tables ... (ver seções 3-6) -->
    <!-- Serviços regionais (S3/IAM/SSM) + legenda na 'region' -->
    <!-- Arestas com parent="1" e <mxGeometry relative="1" as="geometry"/> -->
  </root>
</mxGraphModel>
```

Renderize com:
```
create_diagram(xml=<acima>, routing="libavoid")
```

---

## 11. Checklist de revisão (antes de entregar)

- [ ] Hierarquia de contêineres reflete a realidade (Cloud ⊃ Region ⊃ VPC ⊃
      Subnet ⊃ SG ⊃ recurso) com `parent` correto.
- [ ] Security Groups são molduras tracejadas **em volta** dos recursos, com a
      regra de ingress no rótulo.
- [ ] Ícones `aws4` corretos e cores por categoria coerentes.
- [ ] Route tables presentes, ligadas às gateways com estilo distinto.
- [ ] Subnets multi-AZ: as reservadas para HA estão rotuladas como tal.
- [ ] Toda aresta tem significado, rótulo curto e estilo consistente.
- [ ] Legenda explicando os estilos de linha.
- [ ] IDs/nomes reais nos rótulos (auditável).
- [ ] `routing:"libavoid"`, sem `postLayout` em arquitetura.
- [ ] XML bem-formado: sem comentários, escapes corretos, arestas com geometria.
- [ ] Espaço suficiente (sem sobreposição real); salvo como `.drawio` versionado.

---

## 12. Referências

- AWS Architecture Icons: https://aws.amazon.com/architecture/icons/
- draw.io / mxgraph shape `aws4` (use `search_shapes` para style strings atuais).
- AWS Well-Architected — pilar de Segurança (least privilege, fronteiras de SG).
