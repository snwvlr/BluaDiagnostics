# Política de Segurança

Este é um projeto **acadêmico** desenvolvido pelo nosso grupo na disciplina
de Prompt Engineering e IA da FIAP. Ele **não roda em produção** e **não
processa dados reais de pacientes**.

## Sobre o conteúdo do repositório

- Todos os dados clínicos no projeto são **fictícios e mockados** (ver
  `data/wearables_mock.json` e o histórico simulado dentro do notebook).
- O CPF `12345678901` usado nos exemplos é **inválido por construção**
  (sequência repetida que não passa em validação real).
- Os nomes de pacientes, médicos e protocolos são inventados.
- A "Care Plus" é o cenário do desafio da disciplina; nada que está
  aqui representa o sistema real da operadora.

## Proteção de credenciais

- O notebook carrega a API key do Gemini via **Google Colab Secrets**
  (ou variável de ambiente, ou `getpass` como fallback). Em nenhum dos
  caminhos a chave chega a aparecer no código ou nos outputs.
- O `.gitignore` está configurado pra bloquear arquivos `.env`, `.key`,
  `credentials.json` e similares de entrarem em commit.
- Conferimos antes de cada push que nenhuma chave vazou.

## Se você encontrou alguma chave de API exposta neste repositório

(É bem improvável, mas só por garantia.)

- **Não abra issue pública.** Manda mensagem direta pra qualquer
  integrante do grupo (LinkedIn da FIAP, e-mail institucional) e a
  gente vai:
  1. Revogar a chave imediatamente no Google AI Studio.
  2. Reescrever o histórico do Git pra remover o vazamento.
  3. Gerar uma chave nova e atualizar nosso ambiente local.

## Limitações que a gente reconhece

- O projeto é uma **prova de conceito**. Não foi auditado, não tem
  testes de segurança formais, e não deveria ser usado pra atendimento
  real de pacientes em nenhuma hipótese.
- Os guardrails do agente (não diagnosticar, não prescrever, etc.)
  estão no system prompt e foram testados no nosso eval set, mas
  modelos de linguagem podem falhar de formas inesperadas. Qualquer
  uso real exigiria validação clínica formal, avaliação de viés,
  conformidade com LGPD e supervisão médica contínua, coisas que
  estão **fora do escopo desta disciplina**.
