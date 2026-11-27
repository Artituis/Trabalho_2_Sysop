# Relatório — SimES: simulador de E/S e de sistema de arquivos

**Integrantes:** Artur Santos Lampert

**Repositório:** https://github.com/Artituis/Trabalho_2_Sysop

## 0. Resumo

Este trabalho implementou o subsistema de E/S e operações básicas de sistema de arquivos do SimES, além dos experimentos X1–X5. Foram avaliados escalonadores de disco, polling e interrupções, padrões de acesso, custo de abertura de caminhos e tabelas de arquivos abertos por processo. No X1, o SSTF apresentou a menor resposta média nas cargas testadas, enquanto o SCAN teve vantagens em métricas específicas, como deslocamento e espera máxima em algumas cargas. No X2, polling consumiu mais CPU no HDD, mas o custo estimado de interrupções superou o de polling no dispositivo rápido. No X4, a latência média de abertura aumentou com a profundidade do caminho no modelo sem cache de diretórios. Os resultados descrevem cargas sintéticas e parâmetros fixos, portanto não representam medições diretas de sistemas reais.

## 1. Escopo realizado

| Parte                                                     | Situação  | Observação           |
| --------------------------------------------------------- | --------- | -------------------- |
| N1 · Dispositivo (fornecido)                              | usado     | —                    |
| N2 · Subsistema de E/S (SSTF, SCAN, polling, interrupção) | completo  | [ ]                  |
| N3 · Sistema de arquivos                                  | completo  | [ ]                  |
| N4 · Experimentos X1–X5 e relatório                       | completo  | [ ]                  |
| [M? · nome do módulo]                                     | não feito | Apenas um integrante |
| Bônus flash                                               | não feito | [ ]                  |


Resultado dos testes automáticos (`python3 executar.py testes`): 
```
python3 executar.py testes
test_realocacao (testes.test_fornecido.TesteAlocadorContiguo.test_realocacao) ... ok
test_custo_aleatorio_e_sequencial (testes.test_fornecido.TesteDisco.test_custo_aleatorio_e_sequencial) ... ok
test_exemplo_do_livro (testes.test_fornecido.TesteFCFS.test_exemplo_do_livro) ... ok
test_historico (testes.test_modulos.TesteIntegracao.test_historico) ... skipped 'não implementado: TODO (integração): converter o histórico em requisições'
test_lru_e_fifo (testes.test_modulos.TesteM1Cache.test_lru_e_fifo) ... skipped 'não implementado: TODO (M1): implementar a cache LRU'
test_politicas_de_escrita (testes.test_modulos.TesteM1Cache.test_politicas_de_escrita) ... skipped 'não implementado: TODO (M1): implementar a cache LRU'
test_deadline (testes.test_modulos.TesteM2Escalonadores.test_deadline) ... skipped 'não implementado: TODO (M2): implementar o DEADLINE'
test_livro (testes.test_modulos.TesteM2Escalonadores.test_livro) ... skipped 'não implementado: TODO (M2): implementar o C-SCAN'
test_buffering (testes.test_modulos.TesteM3DMAeBuffering.test_buffering) ... skipped 'não implementado: TODO (M3): simular o buffering'
test_dma (testes.test_modulos.TesteM3DMAeBuffering.test_dma) ... ok
test_links (testes.test_modulos.TesteM4Links.test_links) ... skipped 'não implementado: TODO (M4): link'
test_permissoes_e_travas (testes.test_modulos.TesteM5Protecao.test_permissoes_e_travas) ... skipped 'não implementado: TODO (M5): usuario'
test_custos_de_acesso (testes.test_modulos.TesteM6Alocacao.test_custos_de_acesso) ... skipped 'não implementado: TODO (M6): alocação encadeada'
test_interrupcao (testes.test_nucleo.TesteCPU.test_interrupcao) ... ok
test_polling (testes.test_nucleo.TesteCPU.test_polling) ... ok
test_scan_descendo_livro (testes.test_nucleo.TesteEscalonadores.test_scan_descendo_livro) ... ok
test_scan_subindo (testes.test_nucleo.TesteEscalonadores.test_scan_subindo) ... ok
test_sstf_livro (testes.test_nucleo.TesteEscalonadores.test_sstf_livro) ... ok
test_custos_de_criacao (testes.test_nucleo.TesteSistemaArquivos.test_custos_de_criacao) ... FAIL
test_erros (testes.test_nucleo.TesteSistemaArquivos.test_erros) ... ok
test_escrita_extensao_e_close (testes.test_nucleo.TesteSistemaArquivos.test_escrita_extensao_e_close) ... ok
test_leitura_sequencial (testes.test_nucleo.TesteSistemaArquivos.test_leitura_sequencial) ... ok
test_modo_acrescimo (testes.test_nucleo.TesteSistemaArquivos.test_modo_acrescimo) ... ok
test_resolver_e_open (testes.test_nucleo.TesteSistemaArquivos.test_resolver_e_open) ... ok
test_tabelas_compartilhadas (testes.test_nucleo.TesteSistemaArquivos.test_tabelas_compartilhadas) ... ok
test_unlink_com_arquivo_aberto (testes.test_nucleo.TesteSistemaArquivos.test_unlink_com_arquivo_aberto) ... ok

======================================================================
FAIL: test_custos_de_criacao (testes.test_nucleo.TesteSistemaArquivos.test_custos_de_criacao)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/mnt/c/Users/artur/Documents/Sistemas Operacionais/Trabalho_2_Sysop/testes/test_nucleo.py", line 56, in test_custos_de_criacao
    self.assertEqual(self.custo(self.sa.mkdir, "P1", "/a")[0], (1, 2))
AssertionError: Tuples differ: (0, 2) != (1, 2)

First differing element 0:
0
1

- (0, 2)
?  ^

+ (1, 2)
?  ^


----------------------------------------------------------------------
Ran 26 tests in 0.084s

FAILED (failures=1, skipped=9)
```
## 2. Implementação

### N2 — Subsistema de E/S

**Onde:** 

Polling e interrupção 
Arquivo: `simes/es/cpu.py`
Função: `custo_cpu`


SSTF e SCAN
Arquivo: `simes/es/escalonadores.py`
Função: SSTF.escolher(), e SCAN.escolher()

**Decisões:** 

SSTF, checa qual a requisição mais proxima, como a fila ja está na ordem de chegada a procura com min ja leva em conta o desempate. se ele estiver em outro cilindro troca.

SCAN: primeiro ve se não tem nenhuma requisição que está a frente, se não tiver, passa pelo final e troca de direção, procurando então pela requisição mais próxima.

pooling: demora o tempo da requisição

interrupção: cada bloco de memoria acessado gera uma interrupção que demora o tempo do SO resolver a interrupção e de ler o bloco de memoria.

**Dificuldades:** entender o python e como implementar a solução, isso foi resolvido com o uso da IA.

### N3 — Sistema de arquivos

**Onde:** 

Arquivo: `simes/arquivos/sistema.py`

SistemaArquivos.resolver()
               .mkdir()
               .create()
               .open()
               .read()
               .write()
               .seek()
               .close()
               .unlink()

**Decisões:** [resolução de caminhos, tabelas de arquivos abertos, leitura e escrita, unlink com arquivo aberto]

Resolução de caminhos: caminho é percorrido a partir da raiz, e cada entra da verifica se é um diretório e se tem o FCB checa pra ver se tem links simbólicos.

Tabela de arquivos: esta implementada como na descrição com a posição e o modo.

Leitura: é validado o tamanho pedido e o modo, verifica a trava, lê os blocos que cobrem o intervalo solicitado e avança a posição.

Escrita: valida a quantidade, o handle e o modo; depois garante que há blocos suficientes, grava os blocos que cobrem o intervalo e atualiza posição e metadados.

Unlink: ele não resolve o diretório pai, só remove o link, não o alvo.

**Dificuldades:** implementação e entender como as funções auxiliares funcionavam.

## 3. Experimentos

### X1 — Escalonadores sob quatro padrões de carga

**Hipótese:** FCFS: tempo de resposta será maior pois haverá muitos movimentos redundantes. SSTF: sempre terá o melhor desempenho por ter a menor movimentação. SCAN: é o meio termo entre velocidade e tempo de resposta.

**Fundamentação:** Segundo Silberschatz (seção 10.4), o FCFS atende as requisições na ordem de chegada e essa ordem pode provocar grandes deslocamento da cabeça do disco. O SSTF escolhe a requisição mais próxima da posição atual da cabeça reduzindo o movimento, porém requisições distantes podem esperar mais, aumentando o tempo de espera desses processos. O SCAN percorre o disco em uma direção, atendendo os pedidos ao longo do caminho, e depois vai para o outro lado. Esse modo é mais útil em cargas elevadas e reduz o risco de starvation, mas o desempenho depende da quantidade e padrão das requisições. Por isso espero que o SSTF tenha bons resultados no deslocamento e resposta média e o SCAN ser mais balanceado entre desempenho e espera.

**Configuração:** Deslocamento total, expera máxima e resposta média.

**Resultados:** Cada execução usou 2.000 requisições em um HDD, com interrupção e sem cache. O deslocamento total está em cilindros; os tempos estão em milissegundos.

| Carga | Escalonador | Deslocamento total (cilindros) | Resposta média (ms) | P95 (ms) | Espera máxima (ms) |
|---|---|---:|---:|---:|---:|
| Sequencial | FCFS | 7.184.093 | 10.241,05 | 19.565,45 | 20.598,06 |
| Sequencial | SSTF | 2.336.365 | 17,39 | 32,79 | 37,44 |
| Sequencial | SCAN | 2.469.069 | 23,21 | 43,65 | 48,59 |
| Aleatória | FCFS | 3.316.472 | 50,10 | 145,09 | 193,52 |
| Aleatória | SSTF | 2.631.714 | 20,99 | 48,61 | 116,65 |
| Aleatória | SCAN | 3.372.680 | 31,53 | 65,06 | 116,73 |
| Concentrada | FCFS | 1.864.164 | 19,80 | 48,57 | 96,02 |
| Concentrada | SSTF | 1.613.030 | 15,02 | 37,20 | 130,52 |
| Concentrada | SCAN | 3.338.708 | 33,20 | 68,33 | 118,75 |
| Rajadas | FCFS | 3.341.490 | 266,97 | 505,32 | 597,29 |
| Rajadas | SSTF | 209.811 | 147,49 | 277,72 | 296,99 |
| Rajadas | SCAN | 199.955 | 148,62 | 278,70 | 285,44 |

O SSTF apresentou a menor resposta média nas quatro cargas. Na carga em rajadas, porém, o SCAN teve o menor deslocamento total e a menor espera máxima; o SSTF teve resposta média ligeiramente menor. Assim, o melhor resultado depende da métrica considerada.

**Evidências:** [resultados/es.csv](resultados/es.csv) (registros do experimento X1); [graficos/X1_resposta_media_ms.png](graficos/X1_resposta_media_ms.png); [graficos/X1_deslocamento_total.png](graficos/X1_deslocamento_total.png); [graficos/X1_espera_max_ms.png](graficos/X1_espera_max_ms.png).

![[X1_deslocamento_total.png]]


![[X1_espera_max_ms.png]]


![[X1_resposta_media_ms.png]]


**Análise:** O FCFS teve a maior resposta média nas quatro cargas. A diferença foi especialmente forte na sequencial: 10.241,05 ms, contra 17,39 ms no SSTF e 23,21 ms no SCAN. Essa carga alterna requisições de regiões distantes do disco, o que provoca grandes deslocamentos quando atendidas por ordem de chegada. O SSTF teve a menor resposta média em todas as cargas e o menor deslocamento em três delas. Porém, não foi o melhor em todas as métricas: nas rajadas, o SCAN percorreu menos cilindros e teve espera máxima menor; na carga concentrada, o SSTF teve a maior espera máxima dos três. O SCAN ficou entre FCFS e SSTF em resposta média nas cargas sequencial, aleatória e em rajadas, mas foi o pior na concentrada. Como o SCAN do simulador vai até a borda do disco antes de inverter, esse percurso também elevou seu deslocamento em algumas cargas. Portanto, os dados apoiam a vantagem do SSTF em resposta média, mas não a afirmação de que ele sempre tem o melhor desempenho; o resultado depende da carga e da métrica.

**Veredito:** PARCIALMENTE CONFIRMADA

### X2 — Polling × interrupção conforme o dispositivo

**Hipótese:** O Pooling vai ter muito mais gasto de CPU em um disco rigido do que em um dispositivo rápido, pois a CPU vai ter muito mais buisy waiting pelo disco rigido. Enquanto interrupção talvez tenha mais custo por alta troca de contexto por causa do dispositivo rápido.

**Fundamentação:** Segundo o Silberschatz (seção 13.2), o polling verifica repetidamente o estado do dispositivo até a operação terminar, tendo busy waiting. Com interrupção, o processador executa outras tarefas enquanto o dispositivo externo trabalha, e recebe um sinal quando é pedido a sua atenção, apesar de que cada interrupção tem que ser tratada e tem um custo computacional. Por isso esperasse que o polling consuma mais CPU no HD, que tem latência maior, enquanto no dispositivo rápido esse custo deve diminuir. Como as operações do dispositivo rápido são mais curtas, o custo de tratar interrupções pode representar uma parcela maior do tempo total.

**Configuração:** Computação da CPU que foi gasto em Espera e Saída.

**Resultados:** Cada combinação usou 2.000 requisições com FCFS. A tabela compara a porcentagem de CPU gasta com E/S; a resposta média é incluída para mostrar o efeito do dispositivo. A resposta média é igual entre polling e interrupção em cada combinação.

| Dispositivo | Carga | CPU com polling (%) | CPU com interrupção (%) | Resposta média (ms) |
|---|---|---:|---:|---:|
| HDD | Aleatória | 86,2832 | 1,1213 | 50,1041 |
| HDD | Sequencial | 99,9977 | 0,8361 | 10.241,0482 |
| Rápido | Aleatória | 0,5280 | 1,1219 | 0,0642 |
| Rápido | Sequencial | 1,0739 | 2,2821 | 0,0643 |

**Evidências:** [resultados/es.csv](resultados/es.csv) (registros do experimento X2); [graficos/X2_cpu.png](graficos/X2_cpu.png).

![[X2_cpu.png]]

**Análise:** Totalmente confirmado, o polling usa muito mais cpu quanto se usa memoria lenta, enquanto interrupção tem muito overhead quando se usa memoria mais rápida.

**Veredito:** CONFIRMADA 

### X3 — Acesso sequencial × direto

**Hipótese:** sequencial vai ser mais rápido para vários dados em cadeia, enquanto direto funciona melhor para dados segmentados.

**Fundamentação:** Segundo Silberschatz (seção 11.2), no acesso sequencial os dados são processados em ordem, e a posição do arquivo avança a cada leitura ou escrita. Esse padrão é adequado a tarefas como a compilação, que percorre arquivos de código e cabeçalho. No acesso direto, o programa não precisa percorrer os registros anteriores ao endereço que ele quer, isso favorece leituras de dados aleatórios ou espalhados, como acessar uma entrada em uma tabela de um banco de dados. Assim, cada padrão de acesso tem melhor desempenho dependendo da carga: sequencial na carga do compilador e direto na carga do banco de dados.

**Configuração:** Tempo médio por operação

**Resultados:** As duas cargas foram executadas com alocação contígua e sem erros. O tempo médio por operação ficou em 2,8827 ms para o compilador e 3,0049 ms para o banco de dados. A tabela também informa a quantidade de operações e os acessos médios ao disco; esses valores agregados incluem operações diferentes em cada carga.

| Carga | Operações | Erros | Acessos médios por operação | Tempo médio por operação (ms) |
|---|---:|---:|---:|---:|
| Compilador (sequencial) | 3.904 | 0 | 1,3373 | 2,8827 |
| Banco de dados (direto) | 6.602 | 0 | 0,5463 | 3,0049 |

**Evidências:** [resultados/arquivos.csv](resultados/arquivos.csv) (resumo das cargas X3); [resultados/arquivos_por_operacao.csv](resultados/arquivos_por_operacao.csv) (métricas por operação); [graficos/X3_tempo_por_operacao.png](graficos/X3_tempo_por_operacao.png).

![[X3_tempo_por_operacao.png]]

  **Análise:** Nas operações de leitura, a carga sequencial do compilador teve tempo médio de 1,4828 ms, enquanto a carga aleatória do banco de dados teve 5,4918 ms. Nas escritas, os valores foram 0,9068 ms e 5,5425 ms, respectivamente. Isso é compatível com o custo de buscar posições diferentes no HDD: embora `seek` não gere acesso ao disco neste simulador, a leitura ou escrita na posição escolhida ainda pode exigir o movimento da cabeça. O tempo médio agregado foi próximo (2,8827 ms no compilador e 3,0049 ms no banco), pois cada carga contém operações diferentes. O experimento mostra a vantagem do padrão sequencial para percorrer dados em ordem e o uso do acesso direto para escolher posições sem percorrer os registros anteriores; não testa fragmentação do arquivo.

  **Veredito:** PARCIALMENTE CONFIRMADA. Os dados apoiam a expectativa de menor custo nas leituras e escritas sequenciais, mas não demonstram que o acesso direto seja melhor para arquivos fragmentados, pois a fragmentação não foi variada no X3.


### X4 — Custo de resolver caminhos conforme a profundidade

**Hipótese:** como é uma busca em árvore o custo não deve aumentar muito, talvez linearmente quanto ao número de camadas que ele percorre.

**Fundamentação:** Silberschatz (seção 11.3.5) descreve a organização dos diretórios em árvore, na qual o caminho identifica os diretórios que levam até o arquivo. Para abrir um arquivo, o sistema percorre os componentes e localiza cada entrada. Assim o custo de abrir um arquivo aumenta com a profundidade do caminho. No modelo do simulador cada diretório percorrido acrescenta uma leitura de bloco, portanto esse aumento tende a ser aproximadamente linear com o número de níveis.

**Configuração:** Custo do open conforma a profundidade

**Resultados:** Foram feitas 20 operações de `open` em cada profundidade, sem erros. Cada nível adicional acrescentou uma leitura de diretório por abertura; o tempo médio aumentou de 4,25 ms na profundidade 1 para 4,61 ms na profundidade 10.

| Profundidade | Leituras médias por `open` | Tempo médio de `open` (ms) |
|---:|---:|---:|
| 1 | 2 | 4,25 |
| 2 | 3 | 4,29 |
| 3 | 4 | 4,33 |
| 4 | 5 | 4,37 |
| 5 | 6 | 4,41 |
| 6 | 7 | 4,45 |
| 7 | 8 | 4,49 |
| 8 | 9 | 4,53 |
| 9 | 10 | 4,57 |
| 10 | 11 | 4,61 |

**Evidências:** [resultados/arquivos_por_operacao.csv](resultados/arquivos_por_operacao.csv) (linhas do X4 para `open`); [graficos/X4_open_por_profundidade.png](graficos/X4_open_por_profundidade.png).

![[X4_open_por_profundidade.png]]

**Análise:** mostra claramente como o aumento da profundidade quase não afeta o tempo de abrir o arquivo, por estar estruturado em arvore é muito rápido. eu errei ao dizer que aumentava linearmente, mas sim log de n.

**Veredito:** PARCIALMENTE CONFIRMADA


### X5 — Tabelas de arquivos abertos com vários processos

**Hipótese:** tabela global não seria alterada mas o contador de abertura aumenta.

**Fundamentação:** Silberschatz (seção 11.1.2), nessa seção é descrito o uso de tabelas por processo e de uma tabela global para gerenciar arquivos abertos. A tabela do processo mantém os dados próprios de cada abertura, como a posição atual, e aponta para a entrada correspondente na tabela global. A entrada compartilhada acompanha o arquivo e o número de aberturas. Ela permanece até que o ultimo processo feche o arquivo. Como no X5 vários trabalhadores abrem páginas em comum ao mesmo tempo, espero que cada processo mantenha suas próprias entradas enquanto aberturas da mesma página compartilham a entrada global e aumentam o seu contador.

**Configuração:** Tabelas de arquivos abertas.

**Resultados:** A carga executou 6.279 operações sem erros. A tabela global chegou a 12 entradas, uma por página distinta aberta; cada processo teve no máximo 3 entradas e a soma das tabelas por processo chegou a 24. O contador de aberturas de um mesmo arquivo atingiu 6.

| Métrica | Resultado |
|---|---:|
| Máximo de entradas na tabela global | 12 |
| Máximo de entradas por processo | 3 |
| Máximo da soma das tabelas por processo | 24 |
| Máximo do contador de aberturas por arquivo | 6 |
| Operações | 6.279 |
| Erros | 0 |

**Evidências:** [resultados/arquivos.csv](resultados/arquivos.csv) (resumo do X5); [resultados/arquivos_por_operacao.csv](resultados/arquivos_por_operacao.csv) (operações `open` e `close`); [graficos/X5_tabelas.png](graficos/X5_tabelas.png).

![[X5_tabelas.png]]

**Análise:** A hipótese precisa de uma ressalva. A tabela global chegou a 12 entradas porque até 12 páginas diferentes ficaram abertas ao mesmo tempo; portanto, seu tamanho não permaneceu inalterado durante toda a execução. Porém, cada processo teve no máximo 3 entradas próprias, e aberturas simultâneas da mesma página compartilharam uma única entrada global. O contador dessa entrada chegou a 6, confirmando que novas aberturas do mesmo arquivo incrementam o contador em vez de criar cópias na tabela global. Ao fechar, o contador diminui e a entrada global pode ser removida quando chega a zero.

**Veredito:** PARCIALMENTE CONFIRMADA.


## 4. Relação com sistemas reais

### X1 — Escalonamento de E/S

O X1 compara FCFS, SSTF e SCAN em um disco magnético modelado. Os resultados mostram como a ordem das requisições afeta deslocamento da cabeça, espera e tempo de resposta. Em um sistema real, a política faz parte de uma pilha mais ampla de E/S: no Linux, por exemplo, há escalonadores como `mq-deadline` e BFQ, além da opção `none`, e a escolha disponível depende do dispositivo e da configuração do sistema ([documentação do kernel Linux](https://docs.kernel.org/7.2/block/switching-sched.html)). Portanto, os algoritmos didáticos do experimento ajudam a explicar o compromisso entre desempenho e justiça, mas não reproduzem integralmente os escalonadores atuais nem permitem concluir qual política será melhor em um computador real.

### X2 — Polling e interrupções

O experimento estima o custo de CPU e o tempo de espera de polling e interrupções usando os parâmetros definidos no simulador. Sistemas reais também oferecem operações assíncronas: no Windows, operações sobrepostas permitem que uma thread continue trabalhando enquanto a E/S está em andamento, e portas de conclusão de E/S (IOCP) notificam a conclusão das operações ([operações sobrepostas](https://learn.microsoft.com/en-us/windows/win32/devio/overlapped-operations), [IOCP](https://learn.microsoft.com/en-us/windows/desktop/fileio/i-o-completion-ports)). Essas APIs ilustram que a aplicação pode evitar ficar bloqueada esperando cada operação, mas não equivalem diretamente ao modelo de polling e interrupções do experimento, cujos custos são abstraídos.

### X3 — Padrões de acesso a arquivos

O X3 compara cargas sequenciais e aleatórias em arquivos contíguos. Sistemas reais também podem usar informações sobre o padrão de acesso para otimizar a leitura: no Linux, `posix_fadvise` aceita dicas como acesso sequencial ou aleatório; no Windows, `CreateFile` oferece flags de dica como `FILE_FLAG_SEQUENTIAL_SCAN` e `FILE_FLAG_RANDOM_ACCESS` ([Linux `posix_fadvise(2)`](https://www.man7.org/linux/man-pages/man2/posix_fadvise.2.html), [Windows `CreateFile`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea)). São dicas para otimização, não garantias de desempenho. Como o experimento usa alocação contígua, ele não mede o efeito da fragmentação.

### X4 — Abertura de caminhos e diretórios

No simulador, cada componente do caminho acrescenta uma leitura de diretório ao `open`, por isso caminhos mais profundos elevam o custo estimado. O Linux mantém um cache de nomes de caminhos, chamado dentry cache, que pode atender partes da busca sem consultar novamente o sistema de arquivos ([documentação de busca de caminhos do kernel Linux](https://docs.kernel.org/filesystems/path-lookup.html)). Assim, o crescimento observado no X4 descreve o modelo sem cache; em um sistema real, acessos repetidos podem ser mais baratos. O experimento não mede esse efeito nem compara sistemas de arquivos como ext4, NTFS ou APFS.

### X5 — Abertura compartilhada de arquivos

O X5 representa tabelas por processo e uma tabela global com contador de aberturas. Isso ajuda a visualizar que processos mantêm referências a arquivos abertos, mas é uma abstração. No Linux, cada chamada independente a `open()` cria uma nova descrição de arquivo aberto; descritores duplicados ou herdados por `fork()` podem compartilhar a mesma descrição ([Linux `open(2)`](https://www.man7.org/linux/man-pages/man2/open.2.html)). Portanto, a entrada global única compartilhada no simulador não corresponde exatamente a todas as aberturas independentes do Linux. O experimento demonstra o controle de referências adotado pelo modelo, sem simular todas as regras de compartilhamento e concorrência dos sistemas reais.

## 5. Limitações do simulador

Os experimentos usam cargas sintéticas determinísticas, com parâmetros fixos e sem variação de sementes. Os resultados descrevem essas cargas específicas e não incluem intervalos de confiança nem medem a variação entre execuções.

O disco é representado por tempos fixos de busca, latência rotacional e transferência. O modelo não inclui cache do controlador, detalhes de firmware, concorrência entre dispositivos ou outros efeitos de um HDD real. O dispositivo rápido também é uma configuração simplificada. Assim, os resultados de escalonamento e de custo de polling/interrupção dependem dos parâmetros escolhidos.

O simulador não representa a execução concorrente real da CPU e do disco: o custo de CPU é estimado pelas fórmulas do modelo, sem escalonar processos na CPU em paralelo com a E/S. Portanto, `cpu_es_pct` é uma estimativa relativa ao tempo simulado, não uma medição de utilização de um processador real. No sistema de arquivos, cada diretório ocupa um bloco e não há cache de diretórios; em sistemas reais, caches podem reduzir leituras repetidas. Os experimentos X3–X5 usam alocação contígua, então não permitem tirar conclusões sobre fragmentação ou sobre outros métodos de alocação. O X5 intercala operações de vários processos, mas não simula threads, disputas de execução ou condições de corrida reais. As conclusões deste relatório se limitam aos experimentos X1–X5, executados; X6–X13 não foram implementados nesta entrega.

## 6. Conclusões

1. No X1, o SSTF teve a menor resposta média nas quatro cargas testadas. O melhor escalonador depende também da métrica: nas rajadas, o SCAN teve menor deslocamento total e menor espera máxima, e na carga concentrada o SSTF teve a maior espera máxima.
2. No X2, o polling consumiu muito mais CPU que as interrupções no HDD. No dispositivo rápido, o custo estimado das interrupções foi maior que o do polling, pois o tempo de espera ocupada caiu enquanto o custo de tratamento por bloco permaneceu.
3. No X3, as operações de leitura e escrita sequenciais do compilador foram mais rápidas que as operações aleatórias do banco de dados no HDD. O experimento compara cargas diferentes e não testa fragmentação; portanto, não demonstra que o acesso direto seja melhor para arquivos fragmentados.
4. No X4, cada nível adicional do caminho acrescentou uma leitura de diretório por `open`. O tempo médio aumentou de 4,25 ms na profundidade 1 para 4,61 ms na profundidade 10, conforme esperado pelo modelo.
5. No X5, processos mantiveram entradas próprias e aberturas simultâneas do mesmo arquivo compartilharam a entrada global. O contador de aberturas chegou a 6; a tabela global chegou a 12 entradas ao representar páginas distintas abertas ao mesmo tempo.

## 7. Referências

- SILBERSCHATZ, A.; GALVIN, P. B.; GAGNE, G. *Operating System Concepts*. 9. ed. Wiley, 2013.
- LINUX KERNEL DOCUMENTATION. *Switching I/O schedulers*. Disponível em: [docs.kernel.org](https://docs.kernel.org/7.2/block/switching-sched.html). Acesso em: 26 nov. 2026.
- LINUX KERNEL DOCUMENTATION. *Pathname lookup*. Disponível em: [docs.kernel.org](https://docs.kernel.org/filesystems/path-lookup.html). Acesso em: 26 nov. 2026.
- LINUX MAN-PAGES PROJECT. *open(2)*; *posix_fadvise(2)*. Disponíveis em: [open(2)](https://www.man7.org/linux/man-pages/man2/open.2.html) e [posix_fadvise(2)](https://www.man7.org/linux/man-pages/man2/posix_fadvise.2.html). Acesso em: 26 nov. 2026.
- MICROSOFT. *Overlapped I/O*; *I/O completion ports*; *CreateFile function*. Disponíveis em: [Overlapped operations](https://learn.microsoft.com/en-us/windows/win32/devio/overlapped-operations), [I/O completion ports](https://learn.microsoft.com/en-us/windows/desktop/fileio/i-o-completion-ports) e [CreateFile](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea). Acesso em: 26 nov. 2026.

## 8. Declaração de autoria e uso de IA

**Uso de ferramentas de IA:**

| Ferramenta                | Finalidade                                                                                                                                             | Onde foi usada (arquivos ou seções) |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------- |
| Codex (GPT-6 Luna Medium) | Verificar raciocinio depois de tentativa                                                                                                               | cpu.py                              |
| Codex (GPT-6 Luna Medium) | Criação do código do SSTF e do SCAN                                                                                                                    | escalonadores.py                    |
| Codex (GPT-6 Luna Medium) | criação das funções                                                                                                                                    | sistema.py                          |
| Codex (GPT-6 Luna Medium) | IA gerou um texto para a fundamentação que foi lido e alterado por mim, deixei a seção que ele me deu e alterei o texto pra ficar mais como eu falaria | RELATORIO.md                        |
| Codex (GPT-6 Luna Medium) | Campos de resultado e evidências foram integralmente feitos pela IA                                                                                    | RELATORIO.md                        |
| Codex (GPT-6 Luna Medium) | Analise e veredito do X1, X3 e X5                                                                                                                      | RELATORIO.md                        |
| Codex (GPT-6 Luna Medium) | Seção 0, 4, 5 e 6                                                                                                                                         | RELATORIO.md                        |
