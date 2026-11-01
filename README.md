# PostgreSQL - Rowstore Demo
Author: Prof. Barbosa<br>
Contact: infobarbosa@gmail.com<br>
Github: [infobarbosa](https://github.com/infobarbosa)

## Objetivo
Avaliar de forma rudimentar o comportamento do modelo de armazenamento baseado em linha.<br>
Para isso faremos uso do PostgreSQL pela sua simplicidade e praticidade.

## Ambiente
Este laboratório é executado em uma instância **AWS EC2** (`t3.medium`) provisionada automaticamente via **AWS CloudShell**.<br>
A instância executa um container Docker com o **code-server** (Visual Studio Code direto no navegador) e o **PostgreSQL 16** já embutidos. Todos os comandos do tutorial são executados no terminal do code-server.

> **ATENÇÃO:**
> 1. **Região:** Utilize sempre a região **Norte da Virgínia (`us-east-1`)** no Console AWS.
> 2. **Custos:** A instância EC2 e o volume EBS consomem créditos do seu laboratório AWS Academy. Ao encerrar a aula, garanta que a instância esteja parada (**Stopped**) ou aguarde o desligamento automático da sessão.
> 3. **Não recrie a máquina para continuar o trabalho:** cada execução do `launch-lab.sh` cria uma instância **nova e vazia**, com data e hora no nome. Se a sua instância já existir, siga as instruções de [Retomando o laboratório](#retomando-o-laboratório). Se precisar mesmo recriar, encerre a anterior (**Instance state** > **Terminate instance**) para não consumir créditos.

### Passo 1: Abrir o AWS CloudShell
1. Faça login no Console da AWS através do seu painel do AWS Academy.
2. Certifique-se de que a região selecionada no canto superior direito é **N. Virginia (`us-east-1`)**.
3. No topo do console, clique no ícone do **AWS CloudShell** (`>_`) ao lado da barra de pesquisa.
4. Aguarde o terminal do CloudShell carregar.

### Passo 2: Executar o script de provisionamento
No terminal do CloudShell, execute:
```sh
curl -sS https://raw.githubusercontent.com/infobarbosa/postgres-rowstore-demo/main/launch-lab.sh | bash
```

O script realizará de forma automatizada:
- Descoberta da VPC padrão e de uma Subnet pública.
- Criação do Security Group `lab-postgres-sg` liberando as portas `8080` (IDE) e `22` (SSH).
- Lançamento de uma nova instância EC2 `lab-postgres-rowstore-AAAA-MM-DD_HH_MI_SS` (`t3.medium`, sufixo com a data e hora da execução, Ubuntu 24.04, 20 GB gp3).
- Instalação do Docker e inicialização do container `postgres-lab` (code-server + PostgreSQL).
- Exibição do IP público e do link de acesso.

### Passo 3: Acessar a IDE
1. Abra no navegador o link exibido ao final do script: `http://<IP-PUBLICO>:8080`.
2. Caso a página não responda, **aguarde de 3 a 5 minutos** para que o Docker conclua o download da imagem e atualize o navegador.
3. O code-server abrirá diretamente na IDE, sem solicitar senha.
4. Abra um terminal: menu superior (três linhas horizontais) > **Terminal** > **New Terminal**.

Verifique se o PostgreSQL está no ar:
```sh
psql -c "SELECT version();"

```

Output esperado (a versão e a arquitetura podem variar):
```
                                                                   version                                                                   
---------------------------------------------------------------------------------------------------------------------------------------------
 PostgreSQL 16.15 (Ubuntu 16.15-0ubuntu0.24.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0, 64-bit
(1 row)
```

>### Atenção!
> Os comandos desse tutorial devem ser executados no **terminal do code-server**.

### Retomando o laboratório
Se a sessão do AWS Academy expirar, a instância será parada. **Não execute o `launch-lab.sh` novamente:** ele criaria uma instância nova, sem o seu trabalho.
1. No Console AWS (`us-east-1`), acesse **EC2** > **Instances**.
2. Selecione a sua instância **`lab-postgres-rowstore-<data e hora>`** (se houver mais de uma, a mais recente) e clique em **Instance state** > **Start instance**.
3. Aguarde o status **Running** e copie o novo **Public IPv4 address** (ele muda a cada reinício).
4. Acesse `http://<NOVO_IP_PUBLICO>:8080`. O container reinicia automaticamente e os dados do PostgreSQL são preservados.

### Convenções
Cada bloco de comando do roteiro tem na primeira linha um rótulo, como `# 2.b.1` (seção 2, subseção 2.b, primeiro bloco). Os blocos sem rótulo, logo depois, mostram a saída esperada.

Alguns valores **mudam a cada execução**: o identificador numérico do arquivo (`relfilenode`), os *LSNs* no início da página e, eventualmente, os números de transação (`xmin`/`xmax`). Os seus podem diferir ligeiramente dos do texto, e isso é esperado. Contagens de linhas, tamanhos de tuplas e deslocamentos (`lp_off` / `hex_off`), esses sim, devem bater.

Os comandos funcionam a partir de qualquer pasta. Algumas seções guardam o caminho do arquivo em uma variável (`$ARQUIVO`, `$ARQUIVO_TOAST`); se você abrir um novo terminal no meio do roteiro, rode de novo o bloco que define a variável.

## 1. A base de dados

### 1.a. Database `ecommerce`

Criando a base de dados *ecommerce*:
```sh
# 1.a.1
psql -c "CREATE DATABASE ecommerce;"

```

### 1.b. Tabela `cliente`

Criando a tabela *cliente*:
```sh
# 1.b.1
psql -d ecommerce -c \
"CREATE TABLE cliente(
    id int PRIMARY KEY,
    cpf text,
    nome text
);"

```

Verificando se deu certo:
```sh
# 1.b.2
psql -d ecommerce -c "\d cliente"
```

Output esperado:
```
              Table "public.cliente"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 cpf    | text    |           |          | 
 nome   | text    |           |          | 
Indexes:
    "cliente_pkey" PRIMARY KEY, btree (id)
```

## 2. Operações em linhas (ou registros)

### 2.a. 1o. Insert
```sh
# 2.a.1
psql -d ecommerce -c \
"INSERT INTO cliente(id, cpf, nome)
VALUES (10, '11111111111', 'marcelo barbosa');"

```

Verificando:
```sh
# 2.a.2
psql -d ecommerce -c \
"SELECT * FROM cliente;"
```

Output:
```
 id |     cpf     |      nome       
----+-------------+-----------------
 10 | 11111111111 | marcelo barbosa
(1 row)
```

### 2.b. O arquivo de dados

Diferente de bancos que nomeiam os arquivos com o próprio nome da tabela, o PostgreSQL organiza os arquivos físicos dentro do diretório `$PGDATA/base/<oid_do_banco>/<relfilenode>`. Para descobrir o caminho exato do arquivo da tabela `cliente` e guardá-lo na variável `ARQUIVO`, execute:
```sh
# 2.b.1
ARQUIVO="$PGDATA/$(psql -d ecommerce -tAc "SELECT pg_relation_filepath('cliente');")"
echo "$ARQUIVO"

```

Output esperado (os números no final podem variar):
```
/var/lib/postgresql/16/main/base/16386/16387
```

> **Dica:** O PostgreSQL mantém as páginas recém-alteradas em memória (`shared_buffers`) antes de gravá-las no arquivo de dados da tabela. Neste laboratório, o ambiente força automaticamente um `CHECKPOINT` após comandos de escrita (e a cada 2 segundos em segundo plano). Caso esteja em uma sessão interativa e queira forçar a descarga manualmente, veja a seção [Checkpoint (Flush)](#checkpoint-flush):
> ```
> psql -d ecommerce -c "CHECKPOINT;"
> ```

Verificando o conteúdo do arquivo da tabela `cliente`:
```sh
# 2.b.2
cat "$ARQUIVO"

```

Output:
```
8      ȟp    	 
 11111111111!marcelo barbosa
```

Agora via utilitário `hexdump`:

> O utilitário `hexdump` exibe o conteúdo do arquivo em formato hexadecimal. Na coluna da direita, é possível visualizar os dados legíveis (ASCII).
> No PostgreSQL, o arquivo é dividido em páginas de **8 KB** (`0x2000` = 8192 bytes): o cabeçalho e os ponteiros de linha ficam no início da página (`00000000`), enquanto os registros (tuplas) são gravados **do fim da página (`00002000`) para trás**. Observe como os campos `id` (`0a 00 00 00` = 10), `cpf` e `nome` aparecem **lado a lado**, em sequência — essa é a essência do armazenamento em linha.

```sh
# 2.b.3
hd -C "$ARQUIVO"

```

Output:
```
00000000  00 00 00 00 38 9e d4 01  00 00 00 00 1c 00 c8 1f  |....8...........|
00000010  00 20 04 20 00 00 00 00  c8 9f 70 00 00 00 00 00  |. . ......p.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001fc0  00 00 00 00 00 00 00 00  de 02 00 00 00 00 00 00  |................|
00001fd0  00 00 00 00 00 00 00 00  01 00 03 00 02 09 18 00  |................|
00001fe0  0a 00 00 00 19 31 31 31  31 31 31 31 31 31 31 31  |.....11111111111|
00001ff0  21 6d 61 72 63 65 6c 6f  20 62 61 72 62 6f 73 61  |!marcelo barbosa|
00002000
```

Agora via utilitário `grep`:
```sh
# 2.b.4
grep --text barbosa "$ARQUIVO"

```

Output:
```
 11111111111!marcelo barbosa
```


### 2.c. 2o. Insert
```sh
# 2.c.1
psql -d ecommerce -c \
"INSERT INTO cliente(id, cpf, nome)
VALUES (11, '22222222222', 'Juscelino Kubitschek');"
```

Checando:
```sh
# 2.c.2
psql -d ecommerce -c \
"SELECT * FROM cliente;"
```

Output:
```
 id |     cpf     |         nome         
----+-------------+----------------------
 10 | 11111111111 | marcelo barbosa
 11 | 22222222222 | Juscelino Kubitschek
(2 rows)
```

Verificando o arquivo de dados:

##### `cat`
```sh
# 2.c.3
cat "$ARQUIVO"
```

Output:
```
8      ȟpz    	   22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```sh
# 2.c.4
hd -C "$ARQUIVO"

```

Output:
```
00000000  00 00 00 00 38 a4 d4 01  00 00 00 00 20 00 88 1f  |....8....... ...|
00000010  00 20 04 20 00 00 00 00  c8 9f 70 00 88 9f 7a 00  |. . ......p...z.|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001f80  00 00 00 00 00 00 00 00  df 02 00 00 00 00 00 00  |................|
00001f90  00 00 00 00 00 00 00 00  02 00 03 00 02 09 18 00  |................|
00001fa0  0b 00 00 00 19 32 32 32  32 32 32 32 32 32 32 32  |.....22222222222|
00001fb0  2b 4a 75 73 63 65 6c 69  6e 6f 20 4b 75 62 69 74  |+Juscelino Kubit|
00001fc0  73 63 68 65 6b 00 00 00  de 02 00 00 00 00 00 00  |schek...........|
00001fd0  00 00 00 00 00 00 00 00  01 00 03 00 02 09 18 00  |................|
00001fe0  0a 00 00 00 19 31 31 31  31 31 31 31 31 31 31 31  |.....11111111111|
00001ff0  21 6d 61 72 63 65 6c 6f  20 62 61 72 62 6f 73 61  |!marcelo barbosa|
00002000
```

> Repare que o segundo registro (`Juscelino Kubitschek`, em `00001f88`) foi alocado imediatamente **antes** do primeiro (`marcelo barbosa`, em `00001fc8`), pois as tuplas crescem do fim da página (`00002000`) em direção ao início.

##### `grep`
```sh
# 2.c.5
grep --text Juscelino "$ARQUIVO"

```

Output:
```
8      ȟpz    	   22222222222+Juscelino Kubitschek    	 
```

### 2.d. Insert em lote
```sh
# 2.d.1
psql -d ecommerce -c \
"INSERT INTO cliente (id, cpf, nome) VALUES (1001, '98753936060', 'MARIVALDA KANAMARY');
INSERT INTO cliente (id, cpf, nome) VALUES (1002, '12455426050', 'JUCILENE MOREIRA CRUZ');
INSERT INTO cliente (id, cpf, nome) VALUES (1003, '32487300051', 'GRACIMAR BRASIL GUERRA');
INSERT INTO cliente (id, cpf, nome) VALUES (1004, '59813133074', 'ALDENORA VIANA MOREIRA');
INSERT INTO cliente (id, cpf, nome) VALUES (1005, '79739952003', 'VERA LUCIA RODRIGUES SENA');
INSERT INTO cliente (id, cpf, nome) VALUES (1006, '66142806000', 'IVONE GLAUCIA VIANA DUTRA');
INSERT INTO cliente (id, cpf, nome) VALUES (1007, '19052330000', 'LUCILIA ROSA LIMA PEREIRA');"
```

Verificando se os inserts ocorreram como esperado:
```sh
# 2.d.2
psql -d ecommerce -c \
"SELECT * FROM cliente;"
```

Output esperado:
```
  id  |     cpf     |           nome            
------+-------------+---------------------------
   10 | 11111111111 | marcelo barbosa
   11 | 22222222222 | Juscelino Kubitschek
 1001 | 98753936060 | MARIVALDA KANAMARY
 1002 | 12455426050 | JUCILENE MOREIRA CRUZ
 1003 | 32487300051 | GRACIMAR BRASIL GUERRA
 1004 | 59813133074 | ALDENORA VIANA MOREIRA
 1005 | 79739952003 | VERA LUCIA RODRIGUES SENA
 1006 | 66142806000 | IVONE GLAUCIA VIANA DUTRA
 1007 | 19052330000 | LUCILIA ROSA LIMA PEREIRA
(9 rows)
```

##### `cat`
```sh
# 2.d.3
cat "$ARQUIVO"
```

Output:
```
 <    ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY    	   22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```sh
# 2.d.4
hd -C "$ARQUIVO"

```

Output:
```
00000000  00 00 00 00 e8 aa d4 01  00 00 00 00 3c 00 b0 1d  |............<...|
00000010  00 20 04 20 00 00 00 00  c8 9f 70 00 88 9f 7a 00  |. . ......p...z.|
00000020  48 9f 76 00 08 9f 7c 00  c8 9e 7e 00 88 9e 7e 00  |H.v...|...~...~.|
00000030  40 9e 84 00 f8 9d 84 00  b0 9d 84 00 00 00 00 00  |@...............|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001db0  e0 02 00 00 00 00 00 00  06 00 00 00 00 00 00 00  |................|
00001dc0  09 00 03 00 02 09 18 00  ef 03 00 00 19 31 39 30  |.............190|
00001dd0  35 32 33 33 30 30 30 30  35 4c 55 43 49 4c 49 41  |523300005LUCILIA|
00001de0  20 52 4f 53 41 20 4c 49  4d 41 20 50 45 52 45 49  | ROSA LIMA PEREI|
00001df0  52 41 00 00 00 00 00 00  e0 02 00 00 00 00 00 00  |RA..............|
00001e00  05 00 00 00 00 00 00 00  08 00 03 00 02 09 18 00  |................|
00001e10  ee 03 00 00 19 36 36 31  34 32 38 30 36 30 30 30  |.....66142806000|
00001e20  35 49 56 4f 4e 45 20 47  4c 41 55 43 49 41 20 56  |5IVONE GLAUCIA V|
00001e30  49 41 4e 41 20 44 55 54  52 41 00 00 00 00 00 00  |IANA DUTRA......|
00001e40  e0 02 00 00 00 00 00 00  04 00 00 00 00 00 00 00  |................|
00001e50  07 00 03 00 02 09 18 00  ed 03 00 00 19 37 39 37  |.............797|
00001e60  33 39 39 35 32 30 30 33  35 56 45 52 41 20 4c 55  |399520035VERA LU|
00001e70  43 49 41 20 52 4f 44 52  49 47 55 45 53 20 53 45  |CIA RODRIGUES SE|
00001e80  4e 41 00 00 00 00 00 00  e0 02 00 00 00 00 00 00  |NA..............|
00001e90  03 00 00 00 00 00 00 00  06 00 03 00 02 09 18 00  |................|
00001ea0  ec 03 00 00 19 35 39 38  31 33 31 33 33 30 37 34  |.....59813133074|
00001eb0  2f 41 4c 44 45 4e 4f 52  41 20 56 49 41 4e 41 20  |/ALDENORA VIANA |
00001ec0  4d 4f 52 45 49 52 41 00  e0 02 00 00 00 00 00 00  |MOREIRA.........|
00001ed0  02 00 00 00 00 00 00 00  05 00 03 00 02 09 18 00  |................|
00001ee0  eb 03 00 00 19 33 32 34  38 37 33 30 30 30 35 31  |.....32487300051|
00001ef0  2f 47 52 41 43 49 4d 41  52 20 42 52 41 53 49 4c  |/GRACIMAR BRASIL|
00001f00  20 47 55 45 52 52 41 00  e0 02 00 00 00 00 00 00  | GUERRA.........|
00001f10  01 00 00 00 00 00 00 00  04 00 03 00 02 09 18 00  |................|
00001f20  ea 03 00 00 19 31 32 34  35 35 34 32 36 30 35 30  |.....12455426050|
00001f30  2d 4a 55 43 49 4c 45 4e  45 20 4d 4f 52 45 49 52  |-JUCILENE MOREIR|
00001f40  41 20 43 52 55 5a 00 00  e0 02 00 00 00 00 00 00  |A CRUZ..........|
00001f50  00 00 00 00 00 00 00 00  03 00 03 00 02 09 18 00  |................|
00001f60  e9 03 00 00 19 39 38 37  35 33 39 33 36 30 36 30  |.....98753936060|
00001f70  27 4d 41 52 49 56 41 4c  44 41 20 4b 41 4e 41 4d  |'MARIVALDA KANAM|
00001f80  41 52 59 00 00 00 00 00  df 02 00 00 00 00 00 00  |ARY.............|
00001f90  00 00 00 00 00 00 00 00  02 00 03 00 02 09 18 00  |................|
00001fa0  0b 00 00 00 19 32 32 32  32 32 32 32 32 32 32 32  |.....22222222222|
00001fb0  2b 4a 75 73 63 65 6c 69  6e 6f 20 4b 75 62 69 74  |+Juscelino Kubit|
00001fc0  73 63 68 65 6b 00 00 00  de 02 00 00 00 00 00 00  |schek...........|
00001fd0  00 00 00 00 00 00 00 00  01 00 03 00 02 09 18 00  |................|
00001fe0  0a 00 00 00 19 31 31 31  31 31 31 31 31 31 31 31  |.....11111111111|
00001ff0  21 6d 61 72 63 65 6c 6f  20 62 61 72 62 6f 73 61  |!marcelo barbosa|
00002000
```

##### `grep`
```sh
# 2.d.5
grep --text MARIVALDA "$ARQUIVO"
```

Output:
```
 <    ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY    	   22222222222+Juscelino Kubitschek    	 
```

### 2.e. Delete

Removendo o registro 11 (Juscelino Kubitschek):
```sh
# 2.e.1
psql -d ecommerce -c \
"DELETE FROM cliente WHERE id = 11;"
```

Checando:
```sh
# 2.e.2
psql -d ecommerce -c "SELECT * FROM cliente;"
```

Output:
```
  id  |     cpf     |           nome            
------+-------------+---------------------------
   10 | 11111111111 | marcelo barbosa
 1001 | 98753936060 | MARIVALDA KANAMARY
 1002 | 12455426050 | JUCILENE MOREIRA CRUZ
 1003 | 32487300051 | GRACIMAR BRASIL GUERRA
 1004 | 59813133074 | ALDENORA VIANA MOREIRA
 1005 | 79739952003 | VERA LUCIA RODRIGUES SENA
 1006 | 66142806000 | IVONE GLAUCIA VIANA DUTRA
 1007 | 19052330000 | LUCILIA ROSA LIMA PEREIRA
(8 rows)
```

##### `cat`
```sh
# 2.e.3
cat "$ARQUIVO"

```

Output:
```
Ȯ <     ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```sh
# 2.e.4
hd -C "$ARQUIVO"

```

Output:
```
00000000  00 00 00 00 c8 ae d4 01  00 00 00 00 3c 00 b0 1d  |............<...|
00000010  00 20 04 20 e1 02 00 00  c8 9f 70 00 88 9f 7a 00  |. . ......p...z.|
00000020  48 9f 76 00 08 9f 7c 00  c8 9e 7e 00 88 9e 7e 00  |H.v...|...~...~.|
00000030  40 9e 84 00 f8 9d 84 00  b0 9d 84 00 00 00 00 00  |@...............|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001db0  e0 02 00 00 00 00 00 00  06 00 00 00 00 00 00 00  |................|
00001dc0  09 00 03 00 02 09 18 00  ef 03 00 00 19 31 39 30  |.............190|
00001dd0  35 32 33 33 30 30 30 30  35 4c 55 43 49 4c 49 41  |523300005LUCILIA|
00001de0  20 52 4f 53 41 20 4c 49  4d 41 20 50 45 52 45 49  | ROSA LIMA PEREI|
00001df0  52 41 00 00 00 00 00 00  e0 02 00 00 00 00 00 00  |RA..............|
00001e00  05 00 00 00 00 00 00 00  08 00 03 00 02 09 18 00  |................|
00001e10  ee 03 00 00 19 36 36 31  34 32 38 30 36 30 30 30  |.....66142806000|
00001e20  35 49 56 4f 4e 45 20 47  4c 41 55 43 49 41 20 56  |5IVONE GLAUCIA V|
00001e30  49 41 4e 41 20 44 55 54  52 41 00 00 00 00 00 00  |IANA DUTRA......|
00001e40  e0 02 00 00 00 00 00 00  04 00 00 00 00 00 00 00  |................|
00001e50  07 00 03 00 02 09 18 00  ed 03 00 00 19 37 39 37  |.............797|
00001e60  33 39 39 35 32 30 30 33  35 56 45 52 41 20 4c 55  |399520035VERA LU|
00001e70  43 49 41 20 52 4f 44 52  49 47 55 45 53 20 53 45  |CIA RODRIGUES SE|
00001e80  4e 41 00 00 00 00 00 00  e0 02 00 00 00 00 00 00  |NA..............|
00001e90  03 00 00 00 00 00 00 00  06 00 03 00 02 09 18 00  |................|
00001ea0  ec 03 00 00 19 35 39 38  31 33 31 33 33 30 37 34  |.....59813133074|
00001eb0  2f 41 4c 44 45 4e 4f 52  41 20 56 49 41 4e 41 20  |/ALDENORA VIANA |
00001ec0  4d 4f 52 45 49 52 41 00  e0 02 00 00 00 00 00 00  |MOREIRA.........|
00001ed0  02 00 00 00 00 00 00 00  05 00 03 00 02 09 18 00  |................|
00001ee0  eb 03 00 00 19 33 32 34  38 37 33 30 30 30 35 31  |.....32487300051|
00001ef0  2f 47 52 41 43 49 4d 41  52 20 42 52 41 53 49 4c  |/GRACIMAR BRASIL|
00001f00  20 47 55 45 52 52 41 00  e0 02 00 00 00 00 00 00  | GUERRA.........|
00001f10  01 00 00 00 00 00 00 00  04 00 03 00 02 09 18 00  |................|
00001f20  ea 03 00 00 19 31 32 34  35 35 34 32 36 30 35 30  |.....12455426050|
00001f30  2d 4a 55 43 49 4c 45 4e  45 20 4d 4f 52 45 49 52  |-JUCILENE MOREIR|
00001f40  41 20 43 52 55 5a 00 00  e0 02 00 00 00 00 00 00  |A CRUZ..........|
00001f50  00 00 00 00 00 00 00 00  03 00 03 00 02 09 18 00  |................|
00001f60  e9 03 00 00 19 39 38 37  35 33 39 33 36 30 36 30  |.....98753936060|
00001f70  27 4d 41 52 49 56 41 4c  44 41 20 4b 41 4e 41 4d  |'MARIVALDA KANAM|
00001f80  41 52 59 00 00 00 00 00  df 02 00 00 e1 02 00 00  |ARY.............|
00001f90  00 00 00 00 00 00 00 00  02 00 03 20 02 05 18 00  |........... ....|
00001fa0  0b 00 00 00 19 32 32 32  32 32 32 32 32 32 32 32  |.....22222222222|
00001fb0  2b 4a 75 73 63 65 6c 69  6e 6f 20 4b 75 62 69 74  |+Juscelino Kubit|
00001fc0  73 63 68 65 6b 00 00 00  de 02 00 00 00 00 00 00  |schek...........|
00001fd0  00 00 00 00 00 00 00 00  01 00 03 00 02 09 18 00  |................|
00001fe0  0a 00 00 00 19 31 31 31  31 31 31 31 31 31 31 31  |.....11111111111|
00001ff0  21 6d 61 72 63 65 6c 6f  20 62 61 72 62 6f 73 61  |!marcelo barbosa|
00002000
```

##### `grep`
```sh
# 2.e.5
grep --text Juscelino "$ARQUIVO"

```

Output:
```
Ȯ <     ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
```

> Perceba que, embora o registro tenha sido removido logicamente (não aparece mais no `SELECT`), os dados de `"Juscelino Kubitschek"` **ainda estão presentes no arquivo físico** (em `00001fa0`). Com o controle de concorrência multiversão (**MVCC**), o PostgreSQL apenas preenche o identificador da transação que excluiu a tupla (`xmax` no cabeçalho em `00001f8c`: `e1 02 00 00`), sem apagar os bytes do disco imediatamente.

### 2.f. Update
```sh
# 2.f.1
psql -d ecommerce -c \
"UPDATE cliente SET nome='MARI K.' WHERE id = 1001;"

```

Checando:
```sh
# 2.f.2
psql -d ecommerce -c \
"SELECT * FROM cliente WHERE id = 1001;"

```

Output:
```
  id  |     cpf     |  nome   
------+-------------+---------
 1001 | 98753936060 | MARI K.
(1 row)
```


##### `cat`
```sh
# 2.f.3
cat "$ARQUIVO"
```

Output:
```
 @     ȟpzHv |Ȟ~~@` 
  )   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```sh
# 2.f.4
hd -C "$ARQUIVO"

```

Output:
```
00001d80  e2 02 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001d90  0a 00 03 80 02 29 18 00  e9 03 00 00 19 39 38 37  |.....).......987|
00001da0  35 33 39 33 36 30 36 30  11 4d 41 52 49 20 4b 2e  |53936060.MARI K.|
...
00001f40  41 20 43 52 55 5a 00 00  e0 02 00 00 e2 02 00 00  |A CRUZ..........|
00001f50  00 00 00 00 00 00 00 00  0a 00 03 40 02 05 18 00  |...........@....|
00001f60  e9 03 00 00 19 39 38 37  35 33 39 33 36 30 36 30  |.....98753936060|
00001f70  27 4d 41 52 49 56 41 4c  44 41 20 4b 41 4e 41 4d  |'MARIVALDA KANAM|
00001f80  41 52 59 00 00 00 00 00  df 02 00 00 e1 02 00 00  |ARY.............|
```

##### `grep`
```sh
# 2.f.5
grep --text MARI "$ARQUIVO"
```

Output:
```
  )   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
```

> Perceba uma diferença fundamental do PostgreSQL (MVCC): mesmo reduzindo o tamanho do nome (`'MARIVALDA KANAMARY'` para `'MARI K.'`), o banco **não sobrescreve a tupla no lugar**. Um `UPDATE` no PostgreSQL equivale a marcar a versão antiga como obsoleta (em `00001f60` continua `98753936060'MARIVALDA KANAMARY`) e gravar uma **nova tupla** completa (em `00001d98` surge `98753936060.MARI K.`).

### 2.g. 2o. Update
```sh
# 2.g.1
psql -d ecommerce -c \
"UPDATE cliente SET nome='MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY' WHERE id = 1001;"
```

Checando:
```sh
# 2.g.2
psql -d ecommerce -c "SELECT * FROM cliente WHERE id = 1001;"

```

Output:
```
  id  |     cpf     |                                                                              nome                                                                               
------+-------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 1001 | 98753936060 | MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY
(1 row)
```

##### `cat`
```sh
# 2.g.3
cat "$ARQUIVO"
```

Output:
```
X D     ȟpzHv |Ȟ~~@`     )   98753936060 MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY     %   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```sh
# 2.g.4
hd -C "$ARQUIVO"
```

Output:
```
00000000  00 00 00 00 58 b7 d4 01  00 00 00 00 44 00 b0 1c  |....X.......D...|
00000010  00 20 04 20 e1 02 00 00  c8 9f 70 00 88 9f 7a 00  |. . ......p...z.|
00000020  48 9f 76 00 08 9f 7c 00  c8 9e 7e 00 88 9e 7e 00  |H.v...|...~...~.|
00000030  40 9e 84 00 f8 9d 84 00  b0 9d 84 00 80 9d 60 00  |@.............`.|
00000040  b0 9c a0 01 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001cb0  e3 02 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001cc0  0b 00 03 80 02 29 18 00  e9 03 00 00 19 39 38 37  |.....).......987|
00001cd0  35 33 39 33 36 30 36 30  a0 02 00 00 4d 41 52 49  |53936060....MARI|
00001ce0  56 41 4c 44 41 20 44 45  20 41 4c 43 c3 82 4e 54  |VALDA DE ALC..NT|
00001cf0  41 52 41 20 46 52 41 4e  43 49 53 43 4f 20 41 4e  |ARA FRANCISCO AN|
00001d00  54 c3 94 4e 49 4f 20 4a  4f c3 83 4f 20 43 41 52  |T..NIO JO..O CAR|
00001d10  4c 4f 53 20 58 41 56 49  45 52 20 44 45 20 50 41  |LOS XAVIER DE PA|
00001d20  55 4c 41 20 4d 49 47 55  45 4c 20 52 41 46 41 45  |ULA MIGUEL RAFAE|
00001d30  4c 20 4a 4f 41 51 55 49  4d 20 4a 4f 53 c3 89 20  |L JOAQUIM JOS.. |
00001d40  47 4f 4e 5a 41 47 41 20  50 41 53 43 4f 41 4c 20  |GONZAGA PASCOAL |
00001d50  43 49 50 52 49 41 4e 4f  20 53 45 52 41 46 49 4d  |CIPRIANO SERAFIM|
00001d60  20 44 45 20 42 52 41 47  41 4e c3 87 41 20 45 20  | DE BRAGAN..A E |
00001d70  42 4f 55 52 42 4f 4e 20  4b 41 4e 41 4d 41 52 59  |BOURBON KANAMARY|
00001d80  e2 02 00 00 e3 02 00 00  00 00 00 00 00 00 00 00  |................|
00001d90  0b 00 03 c0 02 25 18 00  e9 03 00 00 19 39 38 37  |.....%.......987|
00001da0  35 33 39 33 36 30 36 30  11 4d 41 52 49 20 4b 2e  |53936060.MARI K.|
00001db0  e0 02 00 00 00 00 00 00  06 00 00 00 00 00 00 00  |................|
00001dc0  09 00 03 00 02 09 18 00  ef 03 00 00 19 31 39 30  |.............190|
00001dd0  35 32 33 33 30 30 30 30  35 4c 55 43 49 4c 49 41  |523300005LUCILIA|
00001de0  20 52 4f 53 41 20 4c 49  4d 41 20 50 45 52 45 49  | ROSA LIMA PEREI|
00001df0  52 41 00 00 00 00 00 00  e0 02 00 00 00 00 00 00  |RA..............|
00001e00  05 00 00 00 00 00 00 00  08 00 03 00 02 09 18 00  |................|
00001e10  ee 03 00 00 19 36 36 31  34 32 38 30 36 30 30 30  |.....66142806000|
00001e20  35 49 56 4f 4e 45 20 47  4c 41 55 43 49 41 20 56  |5IVONE GLAUCIA V|
00001e30  49 41 4e 41 20 44 55 54  52 41 00 00 00 00 00 00  |IANA DUTRA......|
00001e40  e0 02 00 00 00 00 00 00  04 00 00 00 00 00 00 00  |................|
00001e50  07 00 03 00 02 09 18 00  ed 03 00 00 19 37 39 37  |.............797|
00001e60  33 39 39 35 32 30 30 33  35 56 45 52 41 20 4c 55  |399520035VERA LU|
00001e70  43 49 41 20 52 4f 44 52  49 47 55 45 53 20 53 45  |CIA RODRIGUES SE|
00001e80  4e 41 00 00 00 00 00 00  e0 02 00 00 00 00 00 00  |NA..............|
00001e90  03 00 00 00 00 00 00 00  06 00 03 00 02 09 18 00  |................|
00001ea0  ec 03 00 00 19 35 39 38  31 33 31 33 33 30 37 34  |.....59813133074|
00001eb0  2f 41 4c 44 45 4e 4f 52  41 20 56 49 41 4e 41 20  |/ALDENORA VIANA |
00001ec0  4d 4f 52 45 49 52 41 00  e0 02 00 00 00 00 00 00  |MOREIRA.........|
00001ed0  02 00 00 00 00 00 00 00  05 00 03 00 02 09 18 00  |................|
00001ee0  eb 03 00 00 19 33 32 34  38 37 33 30 30 30 35 31  |.....32487300051|
00001ef0  2f 47 52 41 43 49 4d 41  52 20 42 52 41 53 49 4c  |/GRACIMAR BRASIL|
00001f00  20 47 55 45 52 52 41 00  e0 02 00 00 00 00 00 00  | GUERRA.........|
00001f10  01 00 00 00 00 00 00 00  04 00 03 00 02 09 18 00  |................|
00001f20  ea 03 00 00 19 31 32 34  35 35 34 32 36 30 35 30  |.....12455426050|
00001f30  2d 4a 55 43 49 4c 45 4e  45 20 4d 4f 52 45 49 52  |-JUCILENE MOREIR|
00001f40  41 20 43 52 55 5a 00 00  e0 02 00 00 e2 02 00 00  |A CRUZ..........|
00001f50  00 00 00 00 00 00 00 00  0a 00 03 40 02 05 18 00  |...........@....|
00001f60  e9 03 00 00 19 39 38 37  35 33 39 33 36 30 36 30  |.....98753936060|
00001f70  27 4d 41 52 49 56 41 4c  44 41 20 4b 41 4e 41 4d  |'MARIVALDA KANAM|
00001f80  41 52 59 00 00 00 00 00  df 02 00 00 e1 02 00 00  |ARY.............|
00001f90  00 00 00 00 00 00 00 00  02 00 03 20 02 05 18 00  |........... ....|
00001fa0  0b 00 00 00 19 32 32 32  32 32 32 32 32 32 32 32  |.....22222222222|
00001fb0  2b 4a 75 73 63 65 6c 69  6e 6f 20 4b 75 62 69 74  |+Juscelino Kubit|
00001fc0  73 63 68 65 6b 00 00 00  de 02 00 00 00 00 00 00  |schek...........|
00001fd0  00 00 00 00 00 00 00 00  01 00 03 00 02 09 18 00  |................|
00001fe0  0a 00 00 00 19 31 31 31  31 31 31 31 31 31 31 31  |.....11111111111|
00001ff0  21 6d 61 72 63 65 6c 6f  20 62 61 72 62 6f 73 61  |!marcelo barbosa|
00002000
```

##### `grep`
```sh
# 2.g.5
grep --text MARI "$ARQUIVO"
```

Output:
```
X D     ȟpzHv |Ȟ~~@`     )   98753936060 MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY     %   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
```

> Perceba agora que as **três versões** do registro `1001` coexistem fisicamente na página de 8 KB: a original (`MARIVALDA KANAMARY` em `00001f60`), a primeira atualização (`MARI K.` em `00001d98`) e a segunda atualização (`MARIVALDA DE ALCÂNTARA...` em `00001cc8`). As versões antigas (tuplas mortas) permanecem no arquivo até que uma rotina de limpeza (`VACUUM`) libere o espaço para reutilização.


## 3. Por dentro da página e do MVCC

Até aqui inspecionamos os bytes brutos do arquivo em disco. Agora vamos usar os recursos nativos do PostgreSQL para decodificar o cabeçalho de cada linha, os ponteiros da página de 8 KB, a reciclagem de espaço (`VACUUM`) e o armazenamento de atributos grandes (`TOAST`).

### 3.a. Colunas ocultas de sistema (`ctid`, `xmin` e `xmax`)

Toda tabela no PostgreSQL possui colunas ocultas de sistema que revelam o endereçamento físico e o controle de concorrência multiversão (**MVCC**) de cada linha visível:
- **`ctid`:** endereço físico da tupla no formato `(bloco, item)`. O bloco `0` é a primeira página de 8 KB do arquivo, e `item` é o índice do ponteiro de linha (*line pointer*) dentro dessa página.
- **`xmin`:** identificador da transação (*transaction ID*) que inseriu aquela versão da tupla.
- **`xmax`:** identificador da transação que apagou ou substituiu a tupla (`0` indica que a versão atual está ativa).

Consultando as colunas de sistema da tabela `cliente`:
```sh
# 3.a.1
psql -d ecommerce -c \
"SELECT ctid, xmin, xmax, id, cpf, left(nome, 25) AS nome FROM cliente;"
```

Output esperado:
```
  ctid  | xmin | xmax |  id  |     cpf     |           nome            
--------+------+------+------+-------------+---------------------------
 (0,1)  |  734 |    0 |   10 | 11111111111 | marcelo barbosa
 (0,4)  |  736 |    0 | 1002 | 12455426050 | JUCILENE MOREIRA CRUZ
 (0,5)  |  736 |    0 | 1003 | 32487300051 | GRACIMAR BRASIL GUERRA
 (0,6)  |  736 |    0 | 1004 | 59813133074 | ALDENORA VIANA MOREIRA
 (0,7)  |  736 |    0 | 1005 | 79739952003 | VERA LUCIA RODRIGUES SENA
 (0,8)  |  736 |    0 | 1006 | 66142806000 | IVONE GLAUCIA VIANA DUTRA
 (0,9)  |  736 |    0 | 1007 | 19052330000 | LUCILIA ROSA LIMA PEREIRA
 (0,11) |  739 |    0 | 1001 | 98753936060 | MARIVALDA DE ALCÂNTARA FR
(8 rows)
```

> Observe os saltos na coluna `ctid`:
> - O item **`(0,2)`** sumiu do `SELECT` (era `id = 11`, Juscelino Kubitschek, excluído em `2.e`).
> - Os itens **`(0,3)`** e **`(0,10)`** também não aparecem (eram a 1ª e a 2ª versão de `id = 1001`), enquanto a versão atual de `id = 1001` mora agora em **`(0,11)`** com `xmin = 739`.

### 3.b. Anatomia da página de 8 KB com `pageinspect`

Um `SELECT` comum filtra apenas as tuplas visíveis. Para enxergar **todos os 11 itens** gravados na página `0` (incluindo as tuplas mortas) e conectar cada um ao endereço visto no `hexdump`, habilitamos a extensão oficial `pageinspect`:
```sh
# 3.b.1
psql -d ecommerce -c "CREATE EXTENSION pageinspect;"
```

Primeiro, inspecionamos o cabeçalho da página `0` (`page_header`):
```sh
# 3.b.2
psql -d ecommerce -c \
"SELECT lsn, lower, upper, special, pagesize
 FROM page_header(get_raw_page('cliente', 0));"
```

Output:
```
    lsn    | lower | upper | special | pagesize 
-----------+-------+-------+---------+----------
 0/1D6F4E0 |    68 |  7344 |    8192 |     8192
(1 row)
```

> Compare esses números com a primeira linha do `hd -C "$ARQUIVO"` em `2.g.4` (`00000000 ... 44 00 b0 1c`):
> - **`lower = 68`** (`0x0044` em hexadecimal): onde termina o vetor de ponteiros de linha no início da página (24 bytes de cabeçalho + 11 ponteiros de 4 bytes = 68 bytes).
> - **`upper = 7344`** (`0x1cb0` em hexadecimal): onde começa a última tupla gravada no fundo da página (`00001cb0` no `hexdump`).
> - O intervalo entre `68` e `7344` (`7276` bytes preenchidos com `00`) é o espaço livre da página.

Agora listamos todos os 11 itens da página (`heap_page_items`), convertendo o deslocamento `lp_off` para hexadecimal (`hex_off`):
```sh
# 3.b.3
psql -d ecommerce -c \
"SELECT lp, lp_off, to_hex(lp_off::int) AS hex_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
 FROM heap_page_items(get_raw_page('cliente', 0));"
```

Output:
```
 lp | lp_off | hex_off | lp_flags | lp_len | t_xmin | t_xmax | t_ctid 
----+--------+---------+----------+--------+--------+--------+--------
  1 |   8136 | 1fc8    |        1 |     56 |    734 |      0 | (0,1)
  2 |   8072 | 1f88    |        1 |     61 |    735 |    737 | (0,2)
  3 |   8008 | 1f48    |        1 |     59 |    736 |    738 | (0,10)
  4 |   7944 | 1f08    |        1 |     62 |    736 |      0 | (0,4)
  5 |   7880 | 1ec8    |        1 |     63 |    736 |      0 | (0,5)
  6 |   7816 | 1e88    |        1 |     63 |    736 |      0 | (0,6)
  7 |   7744 | 1e40    |        1 |     66 |    736 |      0 | (0,7)
  8 |   7672 | 1df8    |        1 |     66 |    736 |      0 | (0,8)
  9 |   7600 | 1db0    |        1 |     66 |    736 |      0 | (0,9)
 10 |   7552 | 1d80    |        1 |     48 |    738 |    739 | (0,11)
 11 |   7344 | 1cb0    |        1 |    208 |    739 |      0 | (0,11)
(11 rows)
```

> Veja como tudo se encaixa com o `hd -C "$ARQUIVO"`:
> 1. **`hex_off`** é exatamente a posição em bytes de cada tupla no `hexdump` (`1fc8` a primeira linha no fim da página, subindo até `1cb0`).
> 2. **Exclusão (`lp = 2`, Juscelino):** criada pela transação `t_xmin = 735` e excluída pela transação `t_xmax = 737`.
> 3. **Cadeia de versões HOT (`lp = 3 -> 10 -> 11`, Marivalda):**
>    - A versão original em **`(0,3)`** (`hex_off = 1f48`) recebeu `t_xmax = 738` e seu campo `t_ctid` passou a apontar para **`(0,10)`**.
>    - A 2ª versão (`MARI K.`) em **`(0,10)`** (`hex_off = 1d80`) recebeu `t_xmax = 739` e seu `t_ctid` passou a apontar para **`(0,11)`**.
>    - A 3ª versão em **`(0,11)`** (`hex_off = 1cb0`) tem `t_xmax = 0` (ativa) e aponta para si mesma (`(0,11)`). Como a chave primária (`id = 1001`) não mudou e as novas versões couberam na mesma página, o PostgreSQL realizou atualizações **HOT (*Heap-Only Tuple*)**, encadeando as versões dentro da própria página sem precisar criar entradas extras no índice primário.

### 3.c. Limpeza de tuplas mortas: `VACUUM` padrão

Como o PostgreSQL recupera o espaço ocupado pelas tuplas mortas `(0,2)`, `(0,3)` e `(0,10)`? Existem duas formas com comportamentos físicos bem diferentes no disco. Vamos começar pelo `VACUUM` padrão (online, dentro da própria página):
```sh
# 3.c.1
psql -d ecommerce -c "VACUUM cliente;"
```

Verificando o cabeçalho e os itens da página `0` após o `VACUUM`:
```sh
# 3.c.2
psql -d ecommerce -c \
"SELECT lsn, lower, upper, special, pagesize
 FROM page_header(get_raw_page('cliente', 0));"
psql -d ecommerce -c \
"SELECT lp, lp_off, to_hex(lp_off::int) AS hex_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
 FROM heap_page_items(get_raw_page('cliente', 0));"
```

Output:
```
    lsn    | lower | upper | special | pagesize 
-----------+-------+-------+---------+----------
 0/1DCD6C0 |    68 |  7520 |    8192 |     8192
(1 row)

 lp | lp_off | hex_off | lp_flags | lp_len | t_xmin | t_xmax | t_ctid 
----+--------+---------+----------+--------+--------+--------+--------
  1 |   8136 | 1fc8    |        1 |     56 |    734 |      0 | (0,1)
  2 |      0 | 0       |        0 |      0 |        |        | 
  3 |     11 | b       |        2 |      0 |        |        | 
  4 |   8072 | 1f88    |        1 |     62 |    736 |      0 | (0,4)
  5 |   8008 | 1f48    |        1 |     63 |    736 |      0 | (0,5)
  6 |   7944 | 1f08    |        1 |     63 |    736 |      0 | (0,6)
  7 |   7872 | 1ec0    |        1 |     66 |    736 |      0 | (0,7)
  8 |   7800 | 1e78    |        1 |     66 |    736 |      0 | (0,8)
  9 |   7728 | 1e30    |        1 |     66 |    736 |      0 | (0,9)
 10 |      0 | 0       |        0 |      0 |        |        | 
 11 |   7520 | 1d60    |        1 |    208 |    739 |      0 | (0,11)
(11 rows)
```

> Repare no que o `VACUUM` padrão fez **sem bloquear leituras/escritas e sem trocar o arquivo no disco**:
> - Os ponteiros **`lp = 2`** (Juscelino) e **`lp = 10`** (`MARI K.`) foram liberados (`lp_flags = 0`, `LP_UNUSED`).
> - O ponteiro **`lp = 3`** virou um redirecionador (`lp_flags = 2`, `LP_REDIRECT`) apontando diretamente para o item `11` (`lp_off = 11`), pois o índice da chave primária ainda referencia o endereço original `(0,3)`.
> - As tuplas vivas foram compactadas em direção ao fim da página: `upper` subiu de `7344` (`1cb0`) para `7520` (`1d60`), recuperando **176 bytes** de espaço livre dentro da página!

Confirmando com `grep` que os bytes das tuplas mortas foram eliminados na desfragmentação da página:
```sh
# 3.c.3
grep --text Juscelino "$ARQUIVO" || echo "Juscelino removido da pagina"
grep --text "MARI K." "$ARQUIVO" || echo "MARI K. removido da pagina"
```

Output:
```
Juscelino removido da pagina
MARI K. removido da pagina
```

### 3.d. Reescrita física: `VACUUM FULL`

O `VACUUM` padrão organiza o espaço interno das páginas para reutilização pelo próprio banco, mas mantém os ponteiros redirecionados (`lp = 3`) e não devolve páginas vazias do meio da tabela ao sistema operacional. Já o **`VACUUM FULL`** reescreve a tabela inteira do zero em um **novo arquivo físico**:
```sh
# 3.d.1
psql -d ecommerce -c "VACUUM FULL cliente;"
```

Veja o que acontece ao consultar o caminho antigo guardado em `$ARQUIVO`:
```sh
# 3.d.2
ls -la "$ARQUIVO" || true
```

Output:
```
ls: cannot access '/var/lib/postgresql/16/main/base/16386/16387': No such file or directory
```

O arquivo antigo deixou de existir! Atualize a variável `ARQUIVO` com o novo caminho físico da tabela e inspecione os itens da nova página:
```sh
# 3.d.3
ARQUIVO="$PGDATA/$(psql -d ecommerce -tAc "SELECT pg_relation_filepath('cliente');")"
echo "$ARQUIVO"
psql -d ecommerce -c \
"SELECT lp, lp_off, to_hex(lp_off::int) AS hex_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
 FROM heap_page_items(get_raw_page('cliente', 0));"
```

Output:
```
/var/lib/postgresql/16/main/base/16386/16439
 lp | lp_off | hex_off | lp_flags | lp_len | t_xmin | t_xmax | t_ctid 
----+--------+---------+----------+--------+--------+--------+--------
  1 |   8136 | 1fc8    |        1 |     56 |    734 |      0 | (0,1)
  2 |   8072 | 1f88    |        1 |     62 |    736 |      0 | (0,2)
  3 |   8008 | 1f48    |        1 |     63 |    736 |      0 | (0,3)
  4 |   7944 | 1f08    |        1 |     63 |    736 |      0 | (0,4)
  5 |   7872 | 1ec0    |        1 |     66 |    736 |      0 | (0,5)
  6 |   7800 | 1e78    |        1 |     66 |    736 |      0 | (0,6)
  7 |   7728 | 1e30    |        1 |     66 |    736 |      0 | (0,7)
  8 |   7520 | 1d60    |        1 |    208 |    739 |      0 | (0,8)
(8 rows)
```

> No novo arquivo (`16439`), restaram apenas os **8 registros ativos** renumerados sequencialmente de `(0,1)` a `(0,8)`, sem lacunas nem redirecionamentos.

### 3.e. TOAST: compressão inline na própria página

O PostgreSQL não permite que uma única tupla atravesse a fronteira de uma página de **8 KB**. Quando uma tupla ultrapassa cerca de **2 KB**, entra em ação o mecanismo **TOAST (*The Oversized-Attribute Storage Technique*)**, que primeiro tenta comprimir o atributo para mantê-lo na página principal.

Vamos inserir o cliente `2001` cujo `nome` é a string `'POSTGRESQL_ROWSTORE_'` (20 bytes) repetida 150 vezes (**3.000 bytes** lógicos):
```sh
# 3.e.1
psql -d ecommerce -c \
"INSERT INTO cliente (id, cpf, nome)
 VALUES (2001, '88888888888', repeat('POSTGRESQL_ROWSTORE_', 150));"
```

Comparando o tamanho lógico da string (`length`) com os bytes efetivamente ocupados no disco (`pg_column_size`):
```sh
# 3.e.2
psql -d ecommerce -c \
"SELECT id, cpf, length(nome) AS bytes_logicos, pg_column_size(nome) AS bytes_fisicos
 FROM cliente WHERE id = 2001;"
```

Output:
```
  id  |     cpf     | bytes_logicos | bytes_fisicos 
------+-------------+---------------+---------------
 2001 | 88888888888 |          3000 |            65
(1 row)
```

Verificando no `hexdump` como os 3.000 bytes foram gravados em apenas **65 bytes** na página principal (`00001d10` a `00001d55`):
```sh
# 3.e.3
hd -C "$ARQUIVO" | head -n 20
```

Output:
```
00000000  00 00 00 00 98 64 df 01  00 00 00 00 3c 00 f0 1c  |.....d......<...|
00000010  00 20 04 20 00 00 00 00  c8 9f 70 00 88 9f 7c 00  |. . ......p...|.|
00000020  48 9f 7e 00 08 9f 7e 00  c0 9e 84 00 78 9e 84 00  |H.~...~.....x...|
00000030  30 9e 84 00 60 9d a0 01  f0 9c d2 00 00 00 00 00  |0...`...........|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001cf0  e6 02 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001d00  09 00 03 00 02 08 18 00  d1 07 00 00 19 38 38 38  |.............888|
00001d10  38 38 38 38 38 38 38 38  06 01 00 00 b8 0b 00 00  |88888888........|
00001d20  00 50 4f 53 54 47 52 45  53 00 51 4c 5f 52 4f 57  |.POSTGRES.QL_ROW|
00001d30  53 54 f0 4f 52 45 5f 0f  14 ff 0f 14 ff 0f 14 ff  |ST.ORE_.........|
00001d40  0f 14 ff 7f 0f 14 ff 0f  14 ff 0f 14 ff 0f 14 ff  |................|
00001d50  0f 14 ff 0f 14 ff 0f 14  e8 00 00 00 00 00 00 00  |................|
00001d60  e3 02 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001d70  08 00 03 00 02 2b 18 00  e9 03 00 00 19 39 38 37  |.....+.......987|
00001d80  35 33 39 33 36 30 36 30  a0 02 00 00 4d 41 52 49  |53936060....MARI|
00001d90  56 41 4c 44 41 20 44 45  20 41 4c 43 c3 82 4e 54  |VALDA DE ALC..NT|
00001da0  41 52 41 20 46 52 41 4e  43 49 53 43 4f 20 41 4e  |ARA FRANCISCO AN|
00001db0  54 c3 94 4e 49 4f 20 4a  4f c3 83 4f 20 43 41 52  |T..NIO JO..O CAR|
00001dc0  4c 4f 53 20 58 41 56 49  45 52 20 44 45 20 50 41  |LOS XAVIER DE PA|
```

> Observe nas linhas `00001d20` a `00001d50`: o compressor `pglz` do PostgreSQL gravou o padrão `POSTGRESQL_ROWSTORE_` **apenas uma vez** seguido de referências curtas de repetição (`0f 14 ff ...`), mantendo a linha inteira dentro da página de 8 KB.

### 3.f. TOAST: armazenamento externo em `pg_toast`

Se mesmo após a tentativa de compressão o atributo continuar grande, o PostgreSQL move o conteúdo para uma tabela auxiliar (`pg_toast`) dividida em fatias (*chunks*) de ~2 KB, deixando apenas um ponteiro de **18 bytes** na linha original.

Vamos inserir o cliente `2002` com **10.263 bytes** de hashes MD5 concatenados (dados de alta entropia que não comprimem e são maiores que uma página inteira de 8 KB):
```sh
# 3.f.1
psql -d ecommerce -c \
"INSERT INTO cliente (id, cpf, nome)
 SELECT 2002, '99999999999',
        'INICIO_TOAST_' || string_agg(md5(i::text), '') || '_FIM_TOAST'
 FROM generate_series(1, 320) AS i;"
```

Veja o tamanho da tupla `lp = 10` (`id = 2002`) na página principal da tabela `cliente`:
```sh
# 3.f.2
psql -d ecommerce -c \
"SELECT lp, lp_off, to_hex(lp_off::int) AS hex_off, lp_len, t_xmin, t_ctid
 FROM heap_page_items(get_raw_page('cliente', 0))
 WHERE lp >= 9;"
```

Output:
```
 lp | lp_off | hex_off | lp_len | t_xmin | t_ctid 
----+--------+---------+--------+--------+--------
  9 |   7408 | 1cf0    |    105 |    742 | (0,9)
 10 |   7344 | 1cb0    |     58 |    743 | (0,10)
(2 rows)
```

A linha inteira do cliente `2002` ocupa apenas **58 bytes** (`lp_len = 58`) na página principal (24 bytes de cabeçalho + 4 bytes `id` + 12 bytes `cpf` + **18 bytes de ponteiro TOAST**)!

Se procurarmos o texto `INICIO_TOAST` no arquivo principal da tabela `cliente`, ele **não está lá**:
```sh
# 3.f.3
grep --text "INICIO_TOAST" "$ARQUIVO" || echo "Nao encontrado no arquivo principal!"
```

Output:
```
Nao encontrado no arquivo principal!
```

Onde foram parar os 10.263 bytes do `nome`? Na tabela auxiliar **TOAST** associada à tabela `cliente` (`reltoastrelid`). Vamos consultar as fatias (*chunks*) na tabela TOAST e localizar seu arquivo físico em disco:
```sh
# 3.f.4
ARQUIVO_TOAST="$PGDATA/$(psql -d ecommerce -tAc "SELECT pg_relation_filepath(reltoastrelid) FROM pg_class WHERE relname = 'cliente';")"
echo "$ARQUIVO_TOAST"
psql -d ecommerce -c \
"SELECT chunk_id, chunk_seq, length(chunk_data) AS bytes_fatia
 FROM $(psql -d ecommerce -tAc "SELECT reltoastrelid::regclass FROM pg_class WHERE relname = 'cliente';");"
```

Output:
```
/var/lib/postgresql/16/main/base/16386/16442
 chunk_id | chunk_seq | bytes_fatia 
----------+-----------+-------------
    16445 |         0 |        1996
    16445 |         1 |        1996
    16445 |         2 |        1996
    16445 |         3 |        1996
    16445 |         4 |        1996
    16445 |         5 |         283
(6 rows)
```

E buscando no arquivo físico da tabela TOAST (`$ARQUIVO_TOAST`), encontramos o início e o fim do nosso texto de 10 KB:
```sh
# 3.f.5
grep --text -o "INICIO_TOAST_[0-9a-f]\{32\}" "$ARQUIVO_TOAST"
grep --text -o "[0-9a-f]\{32\}_FIM_TOAST" "$ARQUIVO_TOAST"
```

Output:
```
INICIO_TOAST_c4ca4238a0b923820dcc509a6f75849b
320722549d1751cf3f247855f937b982_FIM_TOAST
```


## Parabéns!

Você concluiu com sucesso o laboratório de armazenamento em linha com PostgreSQL! 🎉

Espero que este exercício tenha proporcionado uma compreensão prática sobre o funcionamento do modelo de armazenamento baseado em linha e como o PostgreSQL gerencia páginas de 8 KB, versões de registros (MVCC/HOT), limpeza física (`VACUUM`) e atributos grandes (`TOAST`) no disco.

## Checkpoint (Flush)

Caso necessário, é possível forçar imediatamente a descarga das páginas sujas da memória (`shared_buffers`) para o arquivo de dados em disco executando um `CHECKPOINT`:
```sh
psql -d ecommerce -c "CHECKPOINT;"
```

## Referências

Para se aprofundar nos conceitos abordados neste laboratório:

1. **PostgreSQL Documentation — Database Physical Storage**
   https://www.postgresql.org/docs/16/storage.html

2. **PostgreSQL Documentation — Database Page Layout (`HeapTupleHeaderData`)**
   https://www.postgresql.org/docs/16/storage-page-layout.html

3. **PostgreSQL Documentation — Concurrency Control (MVCC)**
   https://www.postgresql.org/docs/16/mvcc-intro.html

4. **PostgreSQL Documentation — TOAST (*The Oversized-Attribute Storage Technique*)**
   https://www.postgresql.org/docs/16/storage-toast.html

5. **PostgreSQL Documentation — `pageinspect` Extension**
   https://www.postgresql.org/docs/16/pageinspect.html

6. **Row-Oriented vs. Column-Oriented Storage** (conceito geral)
   https://en.wikipedia.org/wiki/Column-oriented_DBMS

7. **Designing Data-Intensive Applications** — Martin Kleppmann
   Capítulo 3: "Storage and Retrieval" — cobre modelos de armazenamento em linha vs. coluna.

## Apêndice: Execução Local

Fora do horário de aula, é possível executar o mesmo ambiente em qualquer máquina (ou provedor de nuvem) com Docker disponível. A imagem é publicada para `linux/amd64` e `linux/arm64` (inclusive Macs com Apple Silicon).

Faça o clone deste repositório e inicialize o container com o `compose.yaml` disponível na raiz do projeto:
```sh
git clone https://github.com/infobarbosa/postgres-rowstore-demo.git
cd postgres-rowstore-demo
docker compose up -d

```

Ou, sem clonar o repositório:
```sh
docker run -d --name postgres-lab -p 8080:8080 ghcr.io/infobarbosa/postgres-lab-docker-image:latest

```

Acesse `http://localhost:8080` no navegador e siga o tutorial a partir do [Passo 3](#passo-3-acessar-a-ide).

Para encerrar e remover o container (os dados do PostgreSQL serão descartados):
```sh
docker compose down

```
ou
```sh
docker rm -f postgres-lab

```
