## 🏗️ Arquitetura de Nuvem (AWS)

Para garantir escalabilidade, persistência de dados e processamento eficiente, a infraestrutura de produção do **Task Manager API** foi desenhada utilizando os serviços de computação em nuvem da **Amazon Web Services (AWS)**. 

O diagrama abaixo ilustra o fluxo de dados e a divisão de responsabilidades entre os componentes:

![Arquitetura AWS](./nome-da-sua-imagem.png) *(Nota: Substitua pelo caminho e nome real da imagem que você exportou do Draw.io)*

---

### 🛠️ Componentes da Infraestrutura e Suas Funções

#### 1. 🖥️ Amazon EC2 (Elastic Compute Cloud)
* **Função:** Servidor de Aplicação Core.
* **Descrição:** Instância virtual Linux responsável por hospedar o ecossistema da aplicação. Através do Docker e Docker Compose, o EC2 roda os containers da API em Spring Boot 3, além da stack de observabilidade (Prometheus e Grafana).

#### 2. 💾 Amazon EBS (Elastic Block Store)
* **Função:** Armazenamento Persistente de Bloco.
* **Descrição:** Funciona como o "disco rígido" de alta performance acoplado diretamente à instância EC2. É utilizado para garantir a persistência estrita dos volumes de dados, como o banco de dados relacional e o histórico de métricas temporais coletadas pelo Prometheus, impedindo a perda de informações caso o servidor seja reiniciado.

#### 3. 🪣 Amazon S3 (Simple Storage Service)
* **Função:** Armazenamento de Objetos de Alta Disponibilidade.
* **Descrição:** Centraliza o armazenamento de arquivos físicos e anexos (como documentos PDF e imagens) enviados pelos usuários nas tarefas. Esta estratégia isola arquivos pesados do banco de dados relacional, reduzindo custos de armazenamento e evitando o overhead no processamento de consultas textuais.

#### 4. ⚡ AWS Lambda
* **Função:** Computação Serverless Orientada a Eventos (*Event-Driven*).
* **Descrição:** Função assíncrona configurada para executar de forma independente. Ela permanece inativa até que um evento ocorra, eliminando custos de ociosidade e liberando a CPU do servidor principal (EC2) para focar estritamente nas requisições HTTP de negócio.

---

### 🔄 Fluxo de Execução e Ciclo de Vida da Requisição

1. **Upload Inicial:** O cliente (Frontend) realiza uma **Requisição REST** do tipo `Multipart/form-data` contendo os dados da tarefa e o arquivo anexo. A requisição é recebida pela API Spring Boot dentro do **Amazon EC2**.
2. **Persistência de Dados Relacionais:** A API processa a regra de negócio e salva as informações textuais estruturadas no banco de dados, cujos arquivos de dados residem no volume seguro do **Amazon EBS**.
3. **Persistência de Arquivos Físicos:** O arquivo binário (imagem ou PDF) é despachado pela API diretamente para o bucket do **Amazon S3**. O banco de dados armazena apenas a String contendo a URL pública/protegida de acesso ao arquivo.
4. **Disparo de Evento Assíncrono:** No momento em que o upload do arquivo é concluído com sucesso, o **Amazon S3** dispara automaticamente um gatilho (*Trigger: Novo Arquivo*) para a **AWS Lambda**. 
5. **Processamento Serverless:** A **AWS Lambda** "acorda" imediatamente para executar tarefas secundárias de segundo plano, como a otimização do tamanho de imagens, geração de miniaturas (thumbnails) ou extração de metadados do documento.

---

### 🚀 Vantagens Arquiteturais Desta Abordagem

* **Desacoplamento Técnico:** A API não gasta poder de processamento compactando arquivos ou salvando binários pesados no banco de dados.
* **Segurança e Isolamento:** Dados críticos ficam no EBS, mídias públicas ficam no S3, e processamentos pesados ficam isolados na Lambda.
* **Otimização de Custos:** O armazenamento no S3 é infinitamente mais barato que expandir discos EBS, e a Lambda só cobra pelos milissegundos exatos em que está processando o arquivo.
