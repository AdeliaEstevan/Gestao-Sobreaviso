# Gestao Sobreaviso Integrada
Sistema web para gestão de escalas de sobreaviso, restrições de disponibilidade e relatórios de acionamento da equipe GEAT/SCADA/ADMS da Energisa, em operação no núcleo de gestão de automação em João Pessoa.
Substituiu um processo manual baseado em planilhas Excel e comunicação por WhatsApp, centralizando cadastros, geração automática da escala mensal, registro de acionamentos com anexo de evidência e comunicação de "De Acordo" entre a fábrica e a gestão.

# Sobre o projeto
O sistema atende duas frentes distintas:
* Fábrica SCADA — os 16 colaboradores da equipe se autenticam, registram restrições de disponibilidade (compromissos pontuais ou férias), visualizam a escala publicada, confirmam ciência da escala do mês e registram relatórios de acionamento com anexo de imagem (print da tela, foto da ocorrência).
* Núcleo de Gestão — administra o cadastro de colaboradores, mantém a agenda de contatos da equipe, gera automaticamente a escala mensal a partir das restrições e das qualificações de sistema (ELIPSE, POWER, ADMS), publica escalas para a fábrica, consulta relatórios de acionamento e acessa os PDFs originais (com imagens embutidas) arquivados no Storage.

#A geração automática da escala respeita regras operacionais reais:

* Limite de 24h de sobreaviso por semana por colaborador
* Finais de semana e feriados como turnos contínuos de 24h
* Dias úteis com turnos fracionados (00:00–07:30, 17:30–07:30 do dia seguinte, 17:30–00:00 na sexta)
* Qualificações por sistema (colaborador só entra em turno de ADMS se estiver habilitado em ADMS)
* Restrições individuais são respeitadas antes da distribuição
* Distribuição balanceada de horas entre os colaboradores

# Recursos

Autenticação e perfis — login por matrícula/senha (PBKDF2-SHA256 no cliente), perfis "fábrica" e "gestão" com módulos e permissões distintos, aceite LGPD no primeiro acesso, log de auditoria centralizado.
Cadastro de colaboradores (gestão) — matrícula, nome, e-mail, telefone corporativo, sistemas em que o colaborador pode operar, perfil de acesso, ativação/desativação.
Restrições de disponibilidade (fábrica) — dois modos: avulso (múltiplos períodos com data, hora início, hora fim e justificativa opcional) e férias (intervalo contínuo de dias). Detecção de sobreposição com restrições já cadastradas, marcação visual de turnos noturnos, edição e exclusão pelo próprio colaborador enquanto o prazo estiver aberto.
Feriados (gestão) — cadastro dos feriados que geram turno de 24h contínuas.
Geração automática da escala (gestão) — algoritmo em dois passos (turnos de 24h primeiro, depois semana a semana) com rollback e recuperação em caso de bloqueio, edição manual via arrastar-e-soltar, preview de distribuição e alertas de desequilíbrio antes da publicação. Suporte a carryover da última semana do mês anterior para não estourar o limite de 24h/semana na transição.
Publicação e "De Acordo" (gestão → fábrica) — a gestão publica a escala do mês; cada colaborador visualiza os próprios turnos e clica em "Estou de acordo" para confirmar ciência. A gestão acompanha em tempo real quem já confirmou.
Relatório de acionamento (fábrica) — formulário para registrar cada acionamento do sobreaviso: sistema envolvido, empresa, data/hora início, data/hora fim, área e nome do solicitante, dados da ocorrência, registro do atendimento, se houve intervenção do SCADA, se outra equipe foi acionada, resolução, observações. Anexo de imagem opcional, PDF timbrado gerado no envio.
Arquivamento de PDFs (Storage) — o PDF completo (com imagem embutida quando houver) é enviado para um bucket privado do Supabase Storage no momento do submit. A gestão visualiza pelo botão "olho" na tabela de acionamentos, com URL assinada de validade curta.
Agenda de telefones (compartilhada) — contatos de outras áreas, terceiros, plantonistas, com número, empresa, cargo, observação e integração com WhatsApp Web.
Exportação — PDF timbrado por acionamento, CSV consolidado de restrições, exportação completa do banco em JSON.

# Stack técnica

* Frontend: HTML, CSS e JavaScript puro em arquivo único, sem bundler, sem framework
* Backend: Supabase (PostgreSQL + Storage + Row-Level Security)
* Autenticação: tabela própria com hash PBKDF2-SHA256 no cliente (não usa Supabase Auth)
* Persistência local: localStorage para cache e rascunho, IndexedDB em ferramentas complementares
* Bibliotecas via CDN: jsPDF (PDF timbrado), Chart.js (gráficos), PapaParse (CSV), Supabase JS SDK
* Hospedagem: GitHub Pages (frontend estático) + Supabase (backend gerenciado)

# Arquitetura e decisões de design

O sistema começou como planilha e foi crescendo em produção sob demanda real. Algumas escolhas de arquitetura merecem contexto: Arquivo único, sem framework nem bundler. É desenvolvido por uma única pessoa (estagiária), com deploy via GitHub Pages e sem pipeline de build. Nesse contexto, o custo de introduzir um framework/bundler seria alto e o ganho, baixo. A modularização é feita via IIFEs e um único módulo ES para o núcleo da gestão. Refatoração para módulos separados está no roadmap se o time crescer.

Supabase com chave anônima e gating no cliente. O sistema não usa Supabase Auth. A tabela de colaboradores mantém o próprio hash de senha, e o RLS libera SELECT/INSERT/UPDATE/DELETE por sessão anon para todas as tabelas. O controle de "quem faz o quê" é feito na camada de UI (a interface da fábrica não expõe telas administrativas). Tradeoff consciente: simplifica o deploy e evita a fricção de rate-limits de e-mail do Supabase Auth durante o cadastro, ao custo de menor defesa contra um cliente malicioso. Adequado ao público interno atual.

PDFs no Storage, texto no Postgres. Imagens em base64 dentro do banco inflam backups, PITR e replicação, e cobram no plano do Postgres. Os relatórios de acionamento gravam apenas o caminho (pdf_path) na linha da tabela acionamentos, e o arquivo em si (com a imagem embutida no PDF) vai para um bucket dedicado no Storage. O acesso é via createSignedUrl de 5 minutos.

Sync em background com guarda contra perda de dados. O cliente puxa dados do Supabase a cada 30 segundos, ao ganhar foco e ao voltar da aba. Enquanto o colaborador está preenchendo um formulário (acionamento ou nova restrição), o re-render é adiado — não apaga o que está digitado. A sincronização retoma quando o formulário é salvo, limpo ou o usuário troca de aba.

Escape consistente contra XSS. Todas as interpolações de dados livres (nome do solicitante, observações, campos digitados) passam por escapeHTML antes de ir para o DOM. Funções de renderização críticas (toasts, tabelas, tooltips, modais) escapam na fonte, protegendo automaticamente todas as chamadas.

# Setup

* 1. Supabase

Criar um projeto novo em supabase.com e rodar uma única vez, via SQL Editor, o script schema_gestao_sobreaviso_completo.sql (incluído no repositório). O script é idempotente: cria as 9 tabelas, os índices, ativa RLS, cria as policies, cria o bucket acionamentos-pdf com suas policies. Pode ser reexecutado sem quebrar dados existentes.

Ao final, verificar:

''' SELECT tablename FROM pg_tables WHERE schemaname='public' ORDER BY tablename;
SELECT id FROM storage.buckets WHERE id='acionamentos-pdf';
SELECT tablename, policyname FROM pg_policies WHERE schemaname='public'; '''

* 2. Configuração da conexão

No topo do arquivo Gestao_Sobreaviso_Integrada_v7.html, ajustar:
const SUPABASE_URL = 'https://SEU-PROJETO.supabase.co';
const SUPABASE_KEY = 'sua-anon-key';

Essas duas linhas são as únicas configurações do frontend.

* 3. Deploy no GitHub Pages
  * Marcar o arquivo HTML como index.html na branch principal (ou apontar o Pages para a raiz)
  * Em Settings → Pages, definir a branch e o diretório
  * O sistema fica acessível em https://SEU-USUARIO.github.io/SEU-REPO/

 Após qualquer atualização, os usuários devem forçar reload (Ctrl+Shift+R) para ignorar o cache.

 * 4. Primeiro acesso
    Ao subir sem dados, o sistema aceita o login inicial gestao / energisa2024 (definido no fluxo de bootstrap). O primeiro passo é cadastrar os colaboradores reais em Núcleo de Gestão → Colaboradores. Depois disso, a senha de gestão deve ser trocada.

# Estrutura do repositório
.
├── Gestao_Sobreaviso_Integrada_v7.html       # Aplicação (frontend inteiro)
├── schema_gestao_sobreaviso_completo.sql     # Schema Supabase (tabelas + RLS + Storage)
├── README.md                                 # Este arquivo
└── docs/                                     # (opcional) documentação adicional

# Uso — guia rápido

Fábrica SCADA — o colaborador faz login, e tem quatro abas: Dashboard (visão geral dos próprios turnos), Nova Restrição (registra períodos avulsos ou férias), Minhas Restrições (edita/exclui restrições enquanto o prazo está aberto), Escala do Mês (visualiza a escala publicada e confirma "Estou de acordo"), Acionamento (registra relatório e anexa imagem).

Núcleo de Gestão — o gestor faz login com perfil de gestão e tem acesso a: Colaboradores (cadastro), Restrições (visualiza e desabilita as recebidas), Feriados, Escala (gera automaticamente, edita por arrastar-e-soltar, publica), Acionamentos (consulta todos os relatórios, exporta PDF, visualiza o PDF original com imagem via botão "olho"), Telefones (agenda compartilhada), Auditoria (log de todas as ações críticas).

# Manutenção

Retenção de PDFs no Storage. O sistema não apaga arquivos automaticamente. A cada ~90 dias, executar no SQL Editor:
'''SELECT name, created_at
  FROM storage.objects
 WHERE bucket_id = 'acionamentos-pdf'
   AND created_at < NOW() - INTERVAL '90 days'
 ORDER BY created_at;'''

Validar visualmente e apagar pela UI do Storage. Depois zerar pdf_path nas linhas correspondentes:

'''UPDATE public.acionamentos
   SET pdf_path = NULL
 WHERE pdf_path IS NOT NULL
   AND criado_em < NOW() - INTERVAL '90 days';'''

Diagnóstico do Storage. Se a gestão relatar que não consegue ver o PDF, abrir o console do navegador (F12) na fábrica logada e rodar:
'''await SupaSync.diagnosticoStorage()'''

O relatório checa cada requisito da migração e reporta o que falta.

Backup manual. A gestão tem no menu um botão "Exportar Tudo" que baixa JSON com colaboradores, restrições, escalas e feriados. Recomendado rodar semanalmente.

# Roadmap
Melhorias mapeadas, em ordem aproximada de prioridade:
* Testes automatizados (extrair funções puras — gerador de escala, validações — para módulo logic.js testável com Vitest)
* Modularização em ES modules (auth.js, escala.js, acionamento.js, sync.js) mantendo GitHub Pages como deploy
* Acessibilidade (labels, aria-*, foco visível, contraste, aria-live para toasts)
* Migração de "De Acordo" para tabela dedicada (elimina race condition do padrão read-modify-write em JSONB)
* Timeout de sessão por inatividade
* Backup agendado via GitHub Actions
* Compressão adicional das imagens antes do embed no PDF

Explicitamente fora do roadmap por agora: migração para React/Vue ou introdução de bundler. O custo em complexidade não justifica o ganho no contexto atual.

# Autoria
Adélia Maria Porpino Estevan — Engenharia Elétrica, estagiária no GEAT/SCADA/ADMS da Energisa Paraíba. Projeto desenvolvido durante o estágio de forma iterativa, guiado por demandas reais da equipe.

# Aviso
Este é um sistema interno da Energisa, desenvolvido para uso da equipe SCADA/ADMS de João Pessoa. O código-fonte é disponibilizado aqui como referência técnica e portfólio da autora. Não constitui produto comercial nem oferece garantia de suporte externo.

Contribuições, sugestões ou correções são bem-vindas via issues neste repositório.
