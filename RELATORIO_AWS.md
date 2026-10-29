# Relatório do projeto — Questionário ABP na AWS

**Status em 29/10/2026:** aplicação adaptada para containers; publicação na conta AWS e capturas de tela pendentes de acesso ao ambiente AWS.

## 1. Aplicação

O projeto é uma aplicação web de questionários. O frontend consiste em páginas HTML, CSS e JavaScript; a API usa Node.js/Express; PostgreSQL guarda usuários, questões e respostas. O esquema e dados iniciais vêm de `abp-1dsm-server/suporte/clausulas_SQL.txt`.

## 2. Arquitetura AWS proposta

```text
Navegador
   │ HTTPS
   ▼
Application Load Balancer (sub-redes públicas)
   │
   ├── ECS Fargate: Nginx + frontend
   └── ECS Fargate: API Node.js
                │ PostgreSQL / rede privada
                ▼
          Amazon RDS PostgreSQL

ECR: imagens dos containers | Secrets Manager: credenciais do banco
CloudWatch Logs: logs da aplicação e serviços
```

O ECS Fargate executa containers sem exigir que a equipe administre servidores EC2. O ECR armazena as imagens Docker. O Application Load Balancer fornece entrada HTTP/HTTPS e distribui solicitações para os serviços. O RDS PostgreSQL mantém os dados fora dos containers, com persistência e backups gerenciados. O banco fica em sub-redes privadas e recebe conexões apenas da tarefa da API. Secrets Manager guarda a senha de conexão; CloudWatch centraliza logs para diagnóstico.

O frontend e a API ficam em serviços ECS separados para permitir atualização e escala independentes. O Nginx serve os arquivos estáticos e encaminha `/api/*` à API. Para HTTPS em produção, associar certificado do AWS Certificate Manager ao Load Balancer e apontar um domínio no Route 53.

## 3. Preparação implementada

- `compose.yaml` define PostgreSQL, API e frontend para execução integrada.
- `abp-1dsm-server/Dockerfile` empacota a API Node.js.
- `abp-1dsm-front/Dockerfile` empacota frontend e Nginx.
- `abp-1dsm-front/nginx.conf` encaminha `/api/*` para a API e `/health` para a verificação de saúde.
- `abp-1dsm-front/js/geral.js` usa `/api` na mesma origem; isso remove a dependência de `localhost` em produção.
- `abp-1dsm-server/src/controladores/bd.js` aceita `BDURI` opcional e SSL selecionável por `BDSSL=true`, ou as variáveis de conexão PostgreSQL individuais.
- A API escuta em `0.0.0.0` e expõe `/health`.

## 4. Execução local e publicação

Para iniciar localmente, com Docker Engine ativo:

```bash
docker compose up --build
```

A aplicação fica disponível em `http://localhost:8080`; a rota de saúde é `http://localhost:8080/health`. Os dados do Postgres persistem no volume `pgdata`. Para reinicializar a base com o SQL inicial, remover o volume (`docker compose down -v`) e iniciar novamente. A senha padrão `abp-local` serve somente para desenvolvimento local; em ambiente AWS, definir uma senha forte pelo Secrets Manager.

Na AWS, construir e publicar as imagens do frontend e API no ECR, criar RDS PostgreSQL em sub-redes privadas, provisionar os serviços ECS Fargate, configurar os grupos de segurança com acesso ao RDS somente pela API, fornecer os segredos pelo Secrets Manager e conectar o ALB ao serviço web. Executar o SQL inicial uma vez no banco de destino antes de liberar a aplicação.

## 5. Evidências solicitadas

As imagens abaixo precisam ser capturadas após a publicação real. Nenhum print de Console AWS foi anexado porque não havia credenciais/sessão AWS disponíveis neste ambiente e não foi possível acessar o daemon Docker. Não apresentar esta versão como implantação já concluída.

1. **AWS — ECS:** tela do cluster e serviços com tarefas em estado `RUNNING`.
2. **AWS — RDS:** instância PostgreSQL em estado `Available` (ocultar endpoint e dados sensíveis).
3. **AWS — Load Balancer:** DNS ou listener do ALB ativo (ocultar identificadores sensíveis, se necessário).
4. **Projeto funcionando:** navegador com a página inicial e uma tela de questionário carregada pela URL pública.

## 6. Estado da entrega

Os arquivos para execução containerizada e a justificativa dos serviços AWS estão neste repositório. A etapa de deploy real, validação no endpoint público e captura das quatro evidências depende de uma sessão AWS autorizada e Docker Engine disponível. Os custos de ECS, ALB e RDS variam por região, dimensionamento e tempo de execução; encerrar recursos de demonstração após a avaliação para evitar cobranças contínuas.
