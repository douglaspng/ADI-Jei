# Segurança

## Autenticação e acesso

- Login com credenciais validadas no servidor; senhas nunca trafegam nem são armazenadas em texto puro no cliente.
- **Rate limiting**: bloqueio temporário após 5 tentativas falhas em 15 minutos, considerando IP e usuário.
- Registro de tentativas de login com limpeza periódica automática.
- Verificação contra bases de senhas vazadas (HIBP) habilitada na autenticação.

## Banco de dados

- **Row Level Security** habilitado em todas as tabelas do domínio.
- Políticas granulares baseadas na identidade do usuário autenticado.
- Tabelas de uso exclusivamente interno (cache de checksum, tentativas de login) operam em modo *default-deny*, acessíveis apenas pelo backend.
- Privilégios de execução em funções administrativas revogados do papel público.

## Edge Functions

- A função de checksum aceita **apenas hosts oficiais da AWS S3 sobre HTTPS**. São bloqueados: endereços IP literais, `localhost`, domínios `.internal`, o endpoint de metadados `169.254.169.254`, credenciais embutidas na URL e portas customizadas — eliminando a superfície de SSRF.
- Credenciais de acesso ao bucket vivem exclusivamente como segredos do ambiente do servidor; nunca são expostas ao navegador.

## Storage

- Bucket privado, sem leitura anônima.
- Políticas de acesso explícitas por operação.

## Frontend

- Preview de XML renderizado como texto puro, prevenindo XSS via conteúdo da planilha.
- Nenhuma chave privada ou segredo no bundle do cliente.

## Processo

- Varreduras automatizadas de segurança da aplicação e de dependências executadas de forma recorrente.
- Vulnerabilidades de cadeia de suprimentos tratadas com atualização imediata das dependências afetadas.
- Achados aceitos conscientemente são documentados com a justificativa técnica.
