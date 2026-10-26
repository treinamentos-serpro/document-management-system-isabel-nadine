# Especificação: Document Management System

**Status:** proposta para orientar a implementação
**Escopo desta entrega:** especificação; nenhuma implementação de backend ou frontend faz parte desta etapa.

## 1. Objetivo

Disponibilizar uma aplicação web para enviar documentos, consultar seus metadados e baixar os arquivos, mantendo os binários no filesystem local e os metadados em memória nesta fase inicial.

## 2. Escopo

### Dentro do escopo

- Envio de um documento por requisição.
- Listagem de documentos e seus metadados.
- Download de um documento pelo identificador.
- Registro do identificador do proprietário no metadado do documento.
- Backend Node.js com Express e frontend React com Vite.
- Armazenamento local dos arquivos em `backend/storage`, usando Multer com `diskStorage`.
- Metadados mantidos em memória durante a execução do processo.
- Comunicação do frontend pelo prefixo `/api`, encaminhado ao backend pelo proxy de desenvolvimento do Vite.

### Fora do escopo

- Provedores externos, armazenamento em nuvem ou serviços de upload de terceiros.
- Banco de dados ou persistência durável dos metadados nesta fase.
- Versionamento de documentos.
- Edição do conteúdo dos arquivos.
- Compartilhamento, busca textual ou organização por pastas.
- Autenticação e autorização completas nesta especificação; a identidade e o isolamento por usuário são decisões pendentes (ver [Decisões pendentes](#10-decisões-pendentes)).

### Estado atual conhecido

O backend é um seed Express com `PORT` configurável e endpoint `GET /health`. Os endpoints documentais definidos nesta especificação ainda não estão implementados. O proxy de desenvolvimento do Vite encaminha `/api/*` para o backend e remove o prefixo `/api` antes do encaminhamento. As camadas `routes`, `controllers`, `services` e `repositories` são a arquitetura prevista para o backend.

`GET /health` é uma verificação operacional existente, não substitui nem faz parte dos requisitos funcionais documentais abaixo.

## 3. Requisitos funcionais

| ID | Requisito | Critério de aceitação |
| --- | --- | --- |
| RF-01 | O sistema deve receber um arquivo por requisição de upload em `multipart/form-data`, no campo `file`. | Com um arquivo válido e as decisões pendentes resolvidas, o sistema armazena o binário localmente, registra os metadados em memória e retorna `201 Created` com o documento criado. |
| RF-02 | O sistema deve rejeitar uploads sem arquivo ou com arquivo que viole as políticas de tipo/tamanho definidas antes da implementação. | Ausência de arquivo retorna `400`; arquivo acima do limite configurado retorna `413`; tipo não permitido retorna `415`. A resposta segue o envelope de erro definido nesta especificação. |
| RF-03 | O sistema deve atribuir um identificador único a cada documento e preservar o nome original como metadado. | Dois uploads distintos não sobrescrevem arquivos existentes; o nome original é retornado como metadado, mas não é usado diretamente como caminho de armazenamento. |
| RF-04 | O sistema deve listar metadados dos documentos visíveis segundo a política de acesso aprovada. | `GET /documents` retorna `200` e uma coleção de metadados; o escopo global ou por proprietário precisa ser decidido antes da implementação. |
| RF-05 | O sistema deve permitir baixar um documento pelo identificador. | Um identificador existente e autorizado retorna o conteúdo como anexo; identificador inexistente ou não visível retorna `404`. |
| RF-06 | O sistema deve informar erros de entrada, ausência de documento e falhas inesperadas em formato consistente. | Toda resposta de erro JSON contém `error.code` e `error.message`, sem expor stack trace, caminhos internos ou detalhes sensíveis. |

## 4. Requisitos não funcionais

| ID | Requisito | Critério de aceitação |
| --- | --- | --- |
| RNF-01 | Os binários devem ser persistidos exclusivamente no filesystem local, em `backend/storage`, com Multer `diskStorage`. | O caminho final é gerado pela aplicação; nenhum provedor remoto é usado. Uma falha ao registrar metadados após gravar o arquivo deve acionar limpeza do arquivo parcial quando possível. |
| RNF-02 | Os metadados devem permanecer em memória nesta fase. | A listagem e a resolução de IDs funcionam enquanto o processo está ativo. Após reinício, os metadados não são garantidos, mesmo que binários permaneçam no diretório. |
| RNF-03 | Configurações operacionais devem ser fornecidas por variáveis de ambiente, seguindo 12-Factor. | `PORT` mantém o valor padrão `3000` quando não configurada; limites futuros de upload, se adotados, devem ter configuração documentada. Segredos não devem ser versionados. |
| RNF-04 | O backend deve seguir dependências unidirecionais da Clean Architecture simples. | O fluxo é `routes -> controllers -> services -> repositories`; services e repositories não dependem de Express ou de componentes React. |
| RNF-05 | Os nomes enviados pelo cliente não devem controlar o caminho físico de armazenamento. | O nome físico deve ser gerado com identificador seguro pela aplicação; o nome original serve apenas para apresentação/download. |
| RNF-06 | Downloads devem ser enviados como anexo, sem interpretação inline pelo navegador. | A resposta usa `Content-Disposition: attachment` e `Content-Type: application/octet-stream`. |
| RNF-07 | A interface deve funcionar em navegadores compatíveis com a versão de React/Vite adotada no projeto e comunicar falhas de upload/listagem/download. | A interface apresenta estados de carregamento, sucesso e erro sem depender de chamadas diretas ao filesystem. |

## 5. Modelo de dados

### Documento (metadados públicos)

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `id` | string | Sim | Identificador único gerado pela aplicação. O formato específico (por exemplo, UUID) deve ser fixado durante a implementação. |
| `originalName` | string | Sim | Nome do arquivo recebido, preservado para exibição e para o nome sugerido no download. Nunca deve ser usado como caminho local. |
| `size` | number | Sim | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Sim | Data e hora do recebimento, em ISO 8601 e UTC. |
| `owner` | string | Sim | Identificador do proprietário associado pelo backend a partir de uma identidade confiável. A origem dessa identidade ainda precisa ser decidida; não confiar em valor arbitrário fornecido pelo cliente. |

### Referência interna de armazenamento

O repositório de arquivos precisa manter uma associação entre `id` e o nome físico gerado no diretório local. Essa referência é interna, não deve ser retornada pela API e pode ser mantida junto aos metadados em memória nesta fase. O arquivo físico não deve ser nomeado diretamente com `originalName`.

### Ciclo de vida e consistência

- Os metadados são voláteis e são perdidos quando o processo reinicia.
- Os binários podem permanecer em `backend/storage` após o reinício; sem metadados, não há garantia de listagem ou download por ID. Reconciliação e limpeza de órfãos não fazem parte desta fase.
- Se a gravação do arquivo for concluída mas a criação do metadado falhar, o backend deve tentar remover o arquivo recém-gravado e retornar erro interno.
- Se o metadado existir, mas o arquivo não puder ser lido, o endpoint de download retorna `404` para o cliente e registra a inconsistência no servidor sem revelar o caminho local.

## 6. Contratos de API

Os caminhos abaixo são os caminhos recebidos pelo Express. No frontend durante desenvolvimento, devem ser chamados com `/api` antes do caminho; o proxy Vite remove esse prefixo. Exemplo: `POST /api/upload` é encaminhado como `POST /upload`.

### Formato de erro comum

Respostas de erro JSON usam `Content-Type: application/json` e o formato:

```json
{
  "error": {
    "code": "FILE_REQUIRED",
    "message": "Envie um arquivo no campo file."
  }
}
```

`message` é uma mensagem segura para o cliente. Detalhes técnicos podem ser registrados no servidor, mas não retornados.

### `POST /upload`

Envia um único arquivo.

- **Content-Type:** `multipart/form-data`.
- **Campo obrigatório:** `file`, contendo um arquivo.
- **Resposta de sucesso:** `201 Created`, `Content-Type: application/json`.

```json
{
  "data": {
    "id": "<identificador>",
    "originalName": "relatorio.pdf",
    "size": 24576,
    "uploadedAt": "2026-10-26T12:00:00.000Z",
    "owner": "<identificador-do-proprietario>"
  }
}
```

- **Erros previstos:**

| Status | Código | Quando ocorre |
| --- | --- | --- |
| `400` | `FILE_REQUIRED` ou `INVALID_UPLOAD` | Campo `file` ausente ou requisição multipart inválida. |
| `413` | `FILE_TOO_LARGE` | Arquivo excede o limite configurado, que deve ser decidido antes da implementação. |
| `415` | `UNSUPPORTED_MEDIA_TYPE` | Tipo de arquivo não aceito segundo a allowlist, que deve ser decidida antes da implementação. |
| `500` | `UPLOAD_FAILED` | Falha inesperada ao gravar arquivo ou registrar metadados. |

Não aceitar mais de um arquivo nessa operação. A política de tipos deve considerar que o MIME informado pelo cliente pode não ser confiável; validação mais forte deve ser escolhida antes da implementação.

### `GET /documents`

Lista metadados; não retorna conteúdo binário.

- **Parâmetros:** nenhum parâmetro de filtro é fixado nesta versão.
- **Resposta de sucesso:** `200 OK`, `Content-Type: application/json`.

```json
{
  "data": [
    {
      "id": "<identificador>",
      "originalName": "relatorio.pdf",
      "size": 24576,
      "uploadedAt": "2026-10-26T12:00:00.000Z",
      "owner": "<identificador-do-proprietario>"
    }
  ]
}
```

A ordenação, paginação e visibilidade dos documentos dependem de decisões registradas na seção 10. A política de acesso precisa ser definida antes de implementar listagem e download; não inferir que a ausência de autenticação torna a listagem global aceitável.

### `GET /documents/:id/download`

Baixa o conteúdo de um documento visível ao solicitante.

- **Parâmetro de caminho:** `id`, identificador do documento.
- **Resposta de sucesso:** `200 OK`, `Content-Type: application/octet-stream`, `Content-Disposition: attachment` com nome sugerido derivado de `originalName` e devidamente tratado pelo framework.
- **Corpo:** bytes do arquivo, sem envelope JSON.
- **Erros previstos:** `404 Not Found` com código `DOCUMENT_NOT_FOUND` quando o documento não existe, não está visível conforme política aprovada, ou o arquivo local não está disponível; `500 Internal Server Error` com código `DOWNLOAD_FAILED` em falha inesperada de leitura.

O endpoint não deve revelar se um documento existe mas pertence a outro usuário, caso seja adotado isolamento por proprietário.

### Códigos HTTP propostos

| Código | Uso |
| --- | --- |
| `200` | Listagem ou download concluído. |
| `201` | Upload e criação de documento concluídos. |
| `400` | Requisição ausente ou inválida. |
| `404` | Documento inexistente ou não visível. |
| `413` | Limite de tamanho excedido. |
| `415` | Tipo não permitido. |
| `500` | Falha inesperada no servidor. |

## 7. Decisões arquiteturais

### Backend

- **Routes:** declaram os caminhos e conectam middlewares/controllers; não implementam regra de negócio.
- **Controllers:** traduzem HTTP para chamadas de serviço, fazem validação básica de entrada e montam status, headers e respostas.
- **Services:** concentram as regras de upload, listagem, visibilidade, download e coordenação de falhas/limpeza.
- **Repositories:** abstraem o filesystem local e a coleção de metadados em memória; não conhecem Express.
- **Multer:** configurado com `diskStorage` para gravar localmente em `backend/storage`; nomes físicos gerados pela aplicação. Limites e filtros devem refletir as decisões pendentes.
- **Inicialização:** manter a configuração Express em CommonJS e respeitar `PORT`; `/health` permanece como endpoint operacional independente.

### Frontend

- React com componentes funcionais e Hooks, organizado nas áreas existentes de `components/`, `pages/` e `services/`.
- Comunicação via `fetch` usando o prefixo `/api` no ambiente de desenvolvimento; tratar erros HTTP, inclusive respostas não JSON no download.
- A interface não acessa o filesystem nem decide identidade/autorização. O backend é a autoridade para metadados e acesso.

### Restrições

- Não adicionar persistência externa, banco de dados, armazenamento em nuvem ou serviço de upload de terceiros.
- Manter dependências e responsabilidades coerentes com a arquitetura já prevista; não mover regra de negócio para routes ou controllers.

## 8. Plano de execução

As etapas abaixo descrevem trabalho futuro. Nesta entrega, somente esta especificação é produzida; nenhuma etapa de código é executada.

1. **Resolver decisões bloqueantes:** aprovar identidade/autenticação e regra de acesso por proprietário; definir tamanho máximo, allowlist de tipos e comportamento de nomes; confirmar política de metadados voláteis. Não iniciar endpoints acessíveis sem resolver como `owner` é estabelecido e quais documentos cada usuário pode ver.
2. **Implementar infraestrutura e upload no backend:** configurar Multer `diskStorage` para `backend/storage`; implementar route, controller, service e repository; gerar ID/nome físico seguro; registrar metadados em memória; mapear erros e limpar arquivo em falha parcial.
3. **Validar upload:** adicionar testes de sucesso, campo ausente, limite/tipo, falha de armazenamento e garantia de não sobrescrever arquivos; manter o armazenamento dos testes isolado e local.
4. **Implementar listagem e download:** aplicar a política aprovada de visibilidade; retornar apenas metadados na listagem; baixar como anexo; mapear IDs inexistentes, arquivos ausentes e falhas de leitura.
5. **Validar API e ciclo de vida:** testar contratos HTTP, headers e códigos; verificar perda de metadados após reinício e documentar o tratamento dos binários órfãos conforme escopo.
6. **Implementar a interface React:** criar os fluxos de envio, listagem e download consumindo `/api`; apresentar estados de carregamento, sucesso e erro; não implementar regras de acesso no cliente.
7. **Integrar e verificar ponta a ponta:** executar backend e frontend com proxy Vite, validar os três fluxos no navegador e confirmar que arquivos ficam apenas no filesystem local e metadados permanecem em memória.

## 9. Critérios de conclusão

- Os três endpoints atendem os contratos desta especificação e aos critérios dos RF-01 a RF-06.
- Os binários são locais, os nomes físicos são controlados pela aplicação e os metadados são mantidos em memória.
- A política de identidade e visibilidade foi decidida antes de habilitar acesso a documentos.
- Os testes cobrem respostas de sucesso, erros previstos, falhas de filesystem e integração via proxy.
- O frontend consegue enviar, listar e baixar documentos e apresenta erros de forma compreensível.
- Nenhum provedor externo ou mecanismo de versionamento foi introduzido.

## 10. Decisões pendentes

Os itens abaixo precisam ser definidos antes da implementação dos fluxos afetados. Os códigos de resposta apresentados são propostas de contrato, não substituem essas decisões.

| ID | Decisão | Impacto |
| --- | --- | --- |
| DP-01 | Como o usuário é autenticado e qual identidade confiável fornece `owner`? | Upload, listagem e download. Não confiar em `owner` enviado livremente pelo cliente. |
| DP-02 | A listagem e o download são globais ou restritos ao proprietário? Como tratar acesso não autorizado? | Autorização e semântica de `GET /documents` e `GET /documents/:id/download`. |
| DP-03 | Quais extensões/MIME são aceitos e qual é o tamanho máximo? | Filtro do Multer, validação, códigos `413`/`415` e configuração. |
| DP-04 | Como evitar colisões de nomes físicos e qual formato final de ID usar? | Repositório local e referência entre metadado e arquivo. A especificação exige nome físico gerado pela aplicação. |
| DP-05 | Qual é a política de ordenação, paginação e volume esperado da listagem? | Tamanho da resposta e desempenho; a primeira versão pode retornar a coleção completa somente após confirmar o volume esperado. |
| DP-06 | Como limpar ou reconciliar arquivos sem metadados após reinício/falha? | Operação do filesystem; limpeza automática está fora do escopo atual e não deve apagar arquivos sem política aprovada. |

## 11. Glossário

- **Binário:** conteúdo físico do arquivo armazenado no filesystem.
- **Metadados:** informações descritivas do documento mantidas em memória.
- **Proprietário (`owner`):** identidade associada ao documento por uma fonte confiável definida pela política de autenticação.
- **Documento visível:** documento que o solicitante pode consultar ou baixar segundo a política de autorização a ser aprovada.
