# Painel Financeiro MC — arquitetura do "contador assistido por IA"

Consultório Dr. Henrique Binato · Juiz de Fora (MG) · Simples Nacional · recebimentos por maquininha e Pix.

## 1. Objetivo e limite

Automatizar tudo o que é operacional na contabilidade do consultório e deixar para um contador com CRC apenas a assinatura e o envio das obrigações formais. O sistema **não** substitui o CRC: PGDAS-D, DCTFWeb, eSocial e DEFIS continuam sendo transmitidos por um profissional habilitado. O que ele faz é chegar nesse profissional com tudo pronto, conferido e auditável, reduzindo o serviço mensal a uma revisão.

## 2. Fatos tributários que moldam o sistema

| Tema | Regra que importa | Fonte |
|---|---|---|
| Regime | Simples Nacional, apuração mensal no PGDAS-D, DAS vence dia 20 do mês seguinte à competência | LC 123/2006; Portal do Simples |
| Atividade médica | Enquadrada no **Anexo V** (LC 123, art. 18, §5º-I). Vai para o **Anexo III** quando o **Fator R ≥ 28%** (§5º-J) | LC 123/2006 |
| Fator R | folha de salários dos últimos 12 meses (pró-labore + salários + encargos) ÷ receita bruta dos últimos 12 meses | Resolução CGSN 140/2018 |
| Impacto | 1ª faixa: 6% no Anexo III contra 15,5% no Anexo V. É a maior alavanca financeira do consultório | tabelas da LC 123 |
| NFS-e | Desde 1º/09/2026 Juiz de Fora emite exclusivamente pelo **Emissor Nacional** (gov.br/nfse). XML no leiaute nacional (`NFSe/infNFSe/DPS/infDPS`) | Decreto municipal 2026; portal PJF |
| ISS | Dentro do DAS para optantes do Simples; o DAM municipal não se aplica | portal PJF |
| Reforma tributária | Empresas do Simples ficam **fora** do ano-teste de CBS/IBS em 2026. Em set/out 2026 abre janela de opção pelo regime híbrido para 2027 (datas divergem entre fontes; confirmar na Resolução CGSN vigente) | LC 214/2025; Resoluções CGSN 186, 190 e 194/2026 |
| Multa PGDAS-D | Desde jan/2026 um dia de atraso já gera multa | Portal do Simples |

Tabelas vigentes usadas no cálculo (receita bruta em 12 meses → alíquota nominal / parcela a deduzir):

| Faixa | RBT12 até | Anexo III | Anexo V |
|---|---|---|---|
| 1 | 180.000 | 6,00% / 0 | 15,50% / 0 |
| 2 | 360.000 | 11,20% / 9.360 | 18,00% / 4.500 |
| 3 | 720.000 | 13,50% / 17.640 | 19,50% / 9.900 |
| 4 | 1.800.000 | 16,00% / 35.640 | 20,50% / 17.100 |
| 5 | 3.600.000 | 21,00% / 125.640 | 23,00% / 62.100 |
| 6 | 4.800.000 | 33,00% / 648.000 | 30,50% / 540.000 |

Alíquota efetiva = (RBT12 × alíquota nominal − parcela a deduzir) ÷ RBT12. DAS do mês = receita do mês × alíquota efetiva.

## 3. Fontes de dados e como entram

| Fonte | Formato | Como o sistema trata |
|---|---|---|
| Banco da clínica (conta PJ) | OFX exportado do internet banking (depois: Open Finance) | Cada transação vira um lançamento. Pix recebido = receita. Crédito de adquirente (Stone, PagSeguro, Rede, Cielo, Getnet…) = **repasse**, não conta como receita para não duplicar com a maquininha |
| Maquininha | CSV/Excel do portal da adquirente | Venda bruta = receita; diferença bruto − líquido = despesa "taxa de maquininha" |
| NFS-e | XML do Emissor Nacional (lote ou unitário); XML ABRASF antigo da PJF também é aceito para o histórico | Cada nota vira um registro com número, competência, tomador e valor. É a **receita de competência** usada no DAS |
| Folha | Pró-labore, salários e encargos categorizados nos lançamentos, ou digitados por mês | Alimenta o Fator R |
| Histórico anterior ao sistema | Receita e folha mensais digitadas | Necessário para fechar a janela de 12 meses do RBT12 |

## 4. Módulos

1. **Importação e normalização** — parsers OFX, CSV (detecção de separador, moeda pt-BR e colunas por cabeçalho) e XML NFS-e. Deduplicação por FITID (banco) e número da nota.
2. **Categorização por regras** — palavras-chave → categoria, editáveis. Categorias têm tipo: receita, despesa, folha ou neutra (transferências, repasses, aplicações).
3. **Conciliação** — casa cada NFS-e com um recebimento (Pix ou venda na maquininha) por valor e janela de datas. Expõe os dois riscos relevantes para médico: **recebimento sem nota** (risco fiscal) e **nota sem recebimento** (cobrança pendente).
4. **Motor do Simples** — RBT12, Fator R, anexo aplicável, alíquota efetiva, DAS estimado, economia do Anexo III e quanto falta de folha para atingir 28%.
5. **Agenda fiscal** — DAS e PGDAS-D (dia 20), DCTFWeb (dia 15), DARF do INSS/IRRF do pró-labore (dia 20), eSocial folha (dia 15), DEFIS (31/03), janela de opção IBS/CBS.
6. **Pacote para o contador** — CSVs de lançamentos e notas, resumo mensal em texto e backup JSON de toda a base.
7. **Assistente** (fase 2) — Claude com acesso aos dados da clínica para responder perguntas sobre os números e sobre tributação, sempre citando a regra aplicada.

## 5. Evolução por fases

| Fase | Entrega | Estado |
|---|---|---|
| 1 | Painel estático (`financeiro/index.html`), dados no navegador, importação manual de OFX/CSV/XML, cálculo do Simples, conciliação, agenda e exportação | **esta versão** |
| 2 | Backend leve (Supabase ou Postgres) para guardar dados fora do navegador e permitir acesso do contador; login | próxima |
| 3 | Integrações automáticas: Open Finance (banco), API da adquirente, consulta de NFS-e no ADN nacional | depois |
| 4 | Assistente Claude sobre os dados; geração do rascunho do PGDAS-D e da folha para o contador apenas conferir e transmitir | depois |

## 6. Decisões de projeto

- **Competência, não caixa.** A receita do DAS vem das notas emitidas no mês, como manda a regra padrão do Simples. Os recebimentos servem para conciliar e detectar receita sem nota.
- **Maquininha pelo bruto.** A nota é emitida pelo valor da consulta; a taxa da adquirente é despesa, nunca redutora de receita.
- **Nada sai do navegador na fase 1.** Dados ficam em `localStorage`; o backup JSON é a responsabilidade do usuário até a fase 2.
- **Tudo auditável.** Cada número do DAS mostra RBT12, folha, Fator R, faixa, alíquota nominal e parcela a deduzir, para o contador conferir em um minuto.

## 7. O que continua humano

- Transmissão de PGDAS-D, DCTFWeb, eSocial e DEFIS (exige CRC).
- Decisão anual sobre o nível de pró-labore: subir a folha até 28% reduz o DAS de 15,5% para 6%, mas o pró-labore paga 11% de INSS e IRPF na pessoa física, enquanto a distribuição de lucros é isenta. O sistema mostra os dois lados; a escolha é do médico com o contador.
- Interpretação de casos ambíguos e defesa em fiscalização.
