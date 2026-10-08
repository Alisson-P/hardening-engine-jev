# Hardening Engine, variante JEV

### O mesmo motor, com uma camada de decisão tipada entre o relatório de conformidade e a correção

[English](README.md) · **Português** | [Plataforma base](https://github.com/Alisson-P/hardening-engine)

> O relatório diz o que está errado. O plano diz o que corrigir primeiro, como,
> e quem precisa estar na sala.

Esta é a prévia pública da **variante JEV** do meu motor de endurecimento de
sistemas. Não é outro motor: a base, o verificador somente leitura e a bancada
são os mesmos. A variante acrescenta uma etapa entre o relatório do verificador
e a correção, a **triagem tipada**, que transforma a lista do que está fora de
conformidade num **plano de correção**, com ordem e caminho para cada achado.

**Comece pelo projeto principal: [hardening-engine](https://github.com/Alisson-P/hardening-engine)**

Aquela página explica a base, a regra de gravidade, o verificador e a bancada.
Esta aqui trata só do que a variante acrescenta.

---

## O vão entre o relatório e a correção

O verificador diz o que está fora de conformidade, e o relatório já ordena por
gravidade. Entre o relatório e a janela de mudança sobra um conjunto de
perguntas que o catálogo sozinho não responde, porque dependem da máquina:

- a leitura mediu mesmo o que o item pede?
- o valor encontrado é tão restritivo quanto o exigido, ou mais, e só a
  comparação literal reprovou?
- esse host usa o componente? Existe controle compensatório? Exceção vigente?
- a correção pode trancar o acesso de quem administra a máquina? Exige
  reinício? Se perde no próximo boot?

Hoje uma pessoa responde isso, achado por achado, e a resposta raramente fica
registrada. A variante faz as mesmas perguntas a um modelo de decisão, uma a
uma, e deixa o código decidir, com limiar escrito antes.

---

## O plano tem duas camadas

**O plano base não precisa de modelo nenhum.** Ele sai só do catálogo:

| Campo | Regra |
|---|---|
| Prioridade | critical P1, high P2, medium P3, low P4 |
| Caminho | script da imagem quando o item é automatizável, verificável e o comando passou na validação; senão, janela de manutenção |
| Trava de acesso | correção que mexe em acesso remoto, autenticação, elevação de privilégio ou firewall vai para o dono do serviço, por regra fixa |

**A camada tipada vem por cima, e só pode fazer três coisas:**

1. **baixar** a prioridade em um degrau, e só com confiança alta
2. **subir** o caminho na escada, nunca descer
3. **acrescentar sinais** para quem vai aplicar a correção

A escada, do menos para o mais humano:

```
script da imagem  <  janela de manutenção  <  dono do serviço  <  revisão de exceção
```

Sem modelo disponível, toda pergunta vira abstenção e o plano sai **igual à
base**. A camada nunca é requisito para o ciclo rodar.

---

## As perguntas

O modelo responde em três formas tipadas, nunca em prosa:

| Primitiva | Pergunta | Devolve |
|---|---|---|
| Noul | Esta afirmação é verdadeira? | uma probabilidade entre 0 e 1 |
| Choice | Qual destas opções? | a opção, a distribuição inteira e a confiança |
| Score | Qual nível nesta rubrica? | o nível, a distribuição inteira e a confiança |

Cada achado recebe **onze perguntas numa requisição só**, e cada item em
conformidade recebe mais uma, própria:

| Grupo | O que pergunta | Primitiva |
|---|---|---|
| O achado está certo? | se a leitura não mede o que o item exige; se o valor encontrado é equivalente ou mais restritivo | Noul |
| Vale para este host? | se o componente está fora do papel do host; se um controle compensatório listado cobre; se uma exceção vigente cobre | Noul |
| O que a correção faz? | se pode trancar o acesso administrativo; se exige reinício; se se perde no próximo boot | Noul |
| Quanto está em jogo? | o risco de quebrar a função do host; a exposição do componente | Score, 4 níveis |
| Quem deve aplicar? | o caminho sugerido, os quatro degraus mais "outro" | Choice, 5 opções |
| Itens em conformidade | se a saída só mostra que algo existe, sem provar o valor | Noul |

O estado que vai com as perguntas leva **só fatos**: o item, a leitura e o que
ela devolveu, e o papel, os serviços, o acesso, a exposição, os controles e as
exceções vigentes do host. **Sem gravidade**, para o modelo não ancorar o
julgamento numa nota que o próprio projeto já deu. Sem nome de máquina, sem
endereço, sem o nome de quem aprovou exceção. A saída da leitura passa por um
saneador que corta em 400 caracteres e mascara hash de senha, endereço IP e
e-mail.

---

## Onde está a decisão

O modelo responde perguntas. Quem compõe as respostas é o **código**, com pesos
e limiares escritos, e cada limiar acompanha o custo do erro.

**Baixar um achado verdadeiro é o erro caro:** a fraqueza fica mais tempo
aberta. Por isso baixar exige confiança 0,70, e nunca mais de um degrau.

**Mandar ao dono uma correção que não precisava é o erro barato:** custa uma
conversa. Por isso os sinais de cautela agem com probabilidade 0,50, sem pedir
confiança.

**A revisão de exceção só vem do grupo de contexto, com confiança alta.** É o
degrau mais humano, e também quer dizer "não corrigir agora", que é o erro caro
de novo. Se o caminho sugerido pedir exceção sem o contexto sustentar, a linha
ganha um sinal e o caminho fica onde estava.

A confiança é calculada no código, com as mesmas fórmulas para qualquer modelo,
então um limiar quer dizer a mesma coisa seja qual for o modelo que responde. O
contrato e as fórmulas seguem a documentação oficial do Jev. Um detalhe pesou
mais do que parece: a confiança do Score é centrada no nível **mais provável**.
Centrar na média ou na mediana parece equivalente e não é. Numa rubrica
respondida como (0; 0,40; 0,25; 0,35; 0), o nível mais provável dá confiança
0,208, a média dá 0,367 e a mediana, 0,375.

---

## As travas

O que nenhuma resposta do modelo atravessa:

1. item critical nunca perde prioridade
2. nenhum achado desce mais de um degrau
3. o caminho nunca desce na escada
4. nenhum achado sai do plano
5. exceção vencida nem chega ao modelo: o código filtra pela data antes
6. para serviço hospedado, o estado sai redigido ou não sai
7. resposta fora do contrato vira abstenção, nunca valor inventado

No fim, a triagem confere o plano inteiro contra a base, e qualquer violação
para tudo: o plano não é gravado. Depois, um leitor separado compara os dois
planos de novo, só pelos arquivos, sem importar nada da triagem.

A triagem nunca escreve na máquina. Ela lê dois arquivos e escreve o plano.

---

## Onde o modelo roda

| Opção | O que é | O estado sai do ambiente? |
|---|---|---|
| Offline | sem modelo, toda resposta vira abstenção, plano igual à base | não |
| Modelo de decisão local | um modelo de decisão aberto servido na própria máquina, o padrão | não |
| Modelo de linguagem local | um modelo compatível com Ollama, probabilidade por autoconsistência | não |
| API hospedada do Jev | o serviço de decisão hospedado, mantido como árbitro de comparação | sim, só redigido |

O modelo de decisão local e a API hospedada falam **o mesmo contrato**, então
um substitui o outro sem mudar uma linha da triagem. A qualidade do modelo
aberto neste domínio **ainda não foi medida**, e o repositório traz um
validador que precisa passar antes de confiar nele.

---

## Como se prova

**Controle de resposta conhecida, 41 de 41.** Cada regra da composição tem um
caso com a resposta certa escrita antes de rodar, inclusive os exemplos
resolvidos da própria documentação do Jev. O gerador dos relatórios do
benchmark também tem caso.

**Regras erradas de propósito, 26 de 26 pegas.** Cada uma estraga uma coisa só
(baixar dois degraus, deixar o caminho descer, centrar o Score na média) e o
controle tem que reprovar. Controle que nunca reprova não prova nada.

**De ponta a ponta, num Ubuntu de verdade.** Um runner descartável do GitHub: o
verificador lê a máquina, o plano sem modelo sai igual à base, o plano com
serviço simulado sai marcado como simulado em toda linha da trilha de
auditoria, e as travas são conferidas de novo, de fora. Verde em menos de um
minuto.

---

## O benchmark

**Leia isto antes dos números.** O tempo de inferência do modelo **não foi
medido**. Quem respondeu foi um serviço simulado, que fala o mesmo contrato e
responde por regra. Então estes números medem o custo do contrato: montar o
estado, transporte, validar a resposta, compor em código, travas e trilha de
auditoria. O tempo do próprio modelo entra por cima disso, e depende de
hardware.

Os relatórios de entrada são sintéticos. Item, leitura, critério e detalhe saem
do catálogo e da própria comparação do motor; só o valor lido é inventado.
Entram só as 59 leituras Linux que dão veredito, e o mesmo item nunca aparece
duas vezes na mesma máquina. Quatro perfis de host, com 100, 1.000 e 5.000
achados cada, numa máquina Linux de um núcleo. Uma requisição por item, faixas
entre os quatro perfis:

| Achados | Média por requisição | p95 | Achados por segundo | Travas violadas |
|---:|---:|---:|---:|---:|
| 100 | 3,5 a 7,5 ms | 4,4 a 39,1 ms | 83 a 179 | 0 |
| 1.000 | 3,2 a 3,6 ms | 4,3 a 7,4 ms | 171 a 190 | 0 |
| 5.000 | 3,2 a 4,8 ms | 5,6 a 16,2 ms | 129 a 191 | 0 |

O custo é por requisição e não cresce com o tamanho do relatório. Com 100
achados, as primeiras chamadas pagam o aquecimento do serviço. Somando as doze
rodadas, foram **24.400 achados** triados, nenhuma trava violada, nenhum
conforme sem uma leitura que mede o item, e a mesma entrada deu o mesmo plano.

**Leia os tempos como indicativos.** A máquina era compartilhada: uma segunda
rodada idêntica deu os mesmos planos e tempos diferentes. A referência mais
limpa é o runner do GitHub, onde, com 200 achados por perfil, o contrato custou
cerca de 1,1 a 1,2 ms por requisição.

**O que a camada muda depende do host.** Com 1.000 achados:

- cada host só atenua o que não é dele: o nó de Kubernetes recebe 17 sinais de
  "contexto atenua" e a estação com interface gráfica 187, contra 204 nos dois
  servidores, porque os itens de permissão de arquivo do Kubernetes são do
  papel do nó e o item do protocolo gráfico moderno é do papel da estação
- o servidor web recebe mais sinais de "exige reinício", 34 contra 17, porque
  as correções que reiniciam o serviço de registro batem num serviço que ele
  presta

O sinal de "risco de quebra alto" saiu igual nos quatro hosts, porque o serviço
simulado julga esse risco pelo item, não pelo host.

**Os caminhos só subiram**, em todas as rodadas. **Nenhum candidato a exceção
apareceu**: com respostas simuladas, o grupo de contexto nunca juntou valor e
confiança suficientes ao mesmo tempo. O caminho existe e o controle prova, mas
este corpus não o exercitou. O outro lado aparece no sinal "modelo sugere
exceção": o caminho sugerido pediu exceção centenas de vezes, e o código não
deixou isso virar caminho sem o contexto sustentar.

> Isto mede quanto a camada custa e o que ela tem permissão de mudar, não se
> ela muda certo. Para virar evidência, falta um modelo de verdade contra uma
> amostra rotulada à mão.

---

## Qual das duas usar

**A base, quando uma pessoa lê o relatório.** O plano base já ordena por
gravidade e protege o acesso por regra. Com algumas dezenas de achados e um
administrador que conhece os hosts, a variante acrescenta pouco.

**A variante, quando o parque passa do que se lê.** Resposta tipada com
confiança calibrada pode ser aplicada por regra: acima do limiar o plano se
ajusta sozinho, abaixo nada muda e o achado fica onde a base pôs. Isso só
funciona porque a confiança é um número comparável.

## O que a variante custa

**Modelo com saída tipada.** O modelo aberto ainda precisa ser medido neste
domínio, e essa é a primeira coisa a validar onde ele vai rodar.

**Uma requisição a mais por item.** Milissegundos para o contrato, mais a
inferência do modelo por cima.

**Mais coisa para manter.** Perguntas, rubricas, pesos e limiares vivem no
repositório, e envelhecem.

## Por que a base continua sendo o projeto principal

O motor funciona **sem inteligência artificial nenhuma**: a base, o
verificador, a bancada e o plano base não dependem de modelo. Quem desligar a
camada continua com um plano completo. O modelo refina a ordem e o caminho, não
sustenta o plano, e nunca toca na máquina.

---

## O que esta prévia mostra, e o que não mostra

**Mostra:** o desenho da triagem, o raciocínio por trás de cada limiar, as
travas e os números.

**Não mostra:** o código, o banco de perguntas, os pesos, o serviço de decisão e
o documento completo do benchmark. Isso tudo fica num repositório privado.

Nenhum dado de ambiente real, de cliente ou de máquina avaliada aparece nesta
prévia. Os relatórios do benchmark são sintéticos, e a rodada de ponta a ponta
usou uma máquina descartável.

Se você quiser ver o conteúdo completo, me chame.

---

## Sobre

Sou Alisson Pereira, trabalho com segurança em nuvem. O motor base diz o que
está errado sem nunca escrever na máquina. Esta variante mantém isso intacto e
acrescenta uma etapa de decisão, onde o modelo responde perguntas pequenas e o
código decide, para que a ordem e o caminho de cada correção possam ser
auditados resposta por resposta.

[LinkedIn](https://www.linkedin.com/in/alisson--pereira/) · [Outros projetos](https://github.com/Alisson-P)

---

## Licença

Esta prévia está sob [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/deed.pt-BR).

Você pode compartilhar dando crédito. Não pode usar comercialmente nem
distribuir versão modificada.
