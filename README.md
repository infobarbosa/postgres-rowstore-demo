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
```
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
```
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

## A base de dados

### Database `ecommerce`

Criando a base de dados *ecommerce*:
```
psql -c "CREATE DATABASE ecommerce;"

```

### Tabela `cliente`

Criando a tabela *cliente*:
```
psql -d ecommerce -c \
"CREATE TABLE cliente(
    id int PRIMARY KEY,
    cpf text,
    nome text
);"

```

Verificando se deu certo:
```
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

## Operações em linhas (ou registros)

### 1. 1o. Insert
```
psql -d ecommerce -c \
"INSERT INTO cliente(id, cpf, nome)
VALUES (10, '11111111111', 'marcelo barbosa');"

```

Verificando:
```
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

### 2. O arquivo de dados

Diferente de bancos que nomeiam os arquivos com o próprio nome da tabela, o PostgreSQL organiza os arquivos físicos dentro do diretório `$PGDATA/base/<oid_do_banco>/<relfilenode>`. Para descobrir o caminho exato do arquivo da tabela `cliente` e guardá-lo na variável `ARQUIVO`, execute:
```
ARQUIVO="$PGDATA/$(psql -d ecommerce -tAc "SELECT pg_relation_filepath('cliente');")"
echo "$ARQUIVO"

```

Output esperado (os números no final podem variar):
```
/var/lib/postgresql/16/main/base/16386/16387
```

> **Dica:** O PostgreSQL mantém as páginas recém-alteradas em memória (`shared_buffers`) antes de gravá-las no arquivo de dados da tabela. Neste laboratório, um processo em segundo plano descarrega automaticamente as alterações para o disco em cerca de **2 segundos**. Se executar um comando de inspeção imediatamente após um `INSERT`, `UPDATE` ou `DELETE` e ainda não vir a alteração, aguarde 2 segundos ou force a gravação imediata conforme a seção [Checkpoint (Flush)](#checkpoint-flush):
> ```
> psql -d ecommerce -c "CHECKPOINT;"
> ```

Verificando o conteúdo do arquivo da tabela `cliente`:
```
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

```
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
```
grep --text barbosa "$ARQUIVO"

```

Output:
```
 11111111111!marcelo barbosa
```


### 3. 2o. Insert
```
psql -d ecommerce -c \
"INSERT INTO cliente(id, cpf, nome)
VALUES (11, '22222222222', 'Juscelino Kubitschek');"
```

Checando:
```
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
```
cat "$ARQUIVO"
```

Output:
```
8      ȟpz    	   22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```
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
```
grep --text Juscelino "$ARQUIVO"

```

Output:
```
8      ȟpz    	   22222222222+Juscelino Kubitschek    	 
```

### 4. Insert em lote
```
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
```
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
```
cat "$ARQUIVO"
```

Output:
```
 <    ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY    	   22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```
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
```
grep --text MARIVALDA "$ARQUIVO"
```

Output:
```
 <    ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY    	   22222222222+Juscelino Kubitschek    	 
```

### 5. Delete

Removendo o registro 11 (Juscelino Kubitschek):
```
psql -d ecommerce -c \
"DELETE FROM cliente WHERE id = 11;"
```

Checando:
```
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
```
cat "$ARQUIVO"

```

Output:
```
Ȯ <     ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```
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
```
grep --text Juscelino "$ARQUIVO"

```

Output:
```
Ȯ <     ȟpzHv |Ȟ~~@  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ    	   98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
```

> Perceba que, embora o registro tenha sido removido logicamente (não aparece mais no `SELECT`), os dados de `"Juscelino Kubitschek"` **ainda estão presentes no arquivo físico** (em `00001fa0`). Com o controle de concorrência multiversão (**MVCC**), o PostgreSQL apenas preenche o identificador da transação que excluiu a tupla (`xmax` no cabeçalho em `00001f8c`: `e1 02 00 00`), sem apagar os bytes do disco imediatamente.

### 6. Update
```
psql -d ecommerce -c \
"UPDATE cliente SET nome='MARI K.' WHERE id = 1001;"

```

Checando:
```
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
```
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
```
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
```
grep --text MARI "$ARQUIVO"
```

Output:
```
  )   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
```

> Perceba uma diferença fundamental do PostgreSQL (MVCC): mesmo reduzindo o tamanho do nome (`'MARIVALDA KANAMARY'` para `'MARI K.'`), o banco **não sobrescreve a tupla no lugar**. Um `UPDATE` no PostgreSQL equivale a marcar a versão antiga como obsoleta (em `00001f60` continua `98753936060'MARIVALDA KANAMARY`) e gravar uma **nova tupla** completa (em `00001d98` surge `98753936060.MARI K.`).

### 7. 2o. Update
```
psql -d ecommerce -c \
"UPDATE cliente SET nome='MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY' WHERE id = 1001;"
```

Checando:
```
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
```
cat "$ARQUIVO"
```

Output:
```
X D     ȟpzHv |Ȟ~~@`     )   98753936060 MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY     %   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
 11111111111!marcelo barbosa
```

##### `hexdump`
```
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
```
grep --text MARI "$ARQUIVO"
```

Output:
```
X D     ȟpzHv |Ȟ~~@`     )   98753936060 MARIVALDA DE ALCÂNTARA FRANCISCO ANTÔNIO JOÃO CARLOS XAVIER DE PAULA MIGUEL RAFAEL JOAQUIM JOSÉ GONZAGA PASCOAL CIPRIANO SERAFIM DE BRAGANÇA E BOURBON KANAMARY     %   98753936060 MARI K.  	  	   190523300005LUCILIA ROSA LIMA PEREIRA     	   661428060005IVONE GLAUCIA VIANA DUTRA     	   797399520035VERA LUCIA RODRIGUES SENA     	   59813133074/ALDENORA VIANA MOREIRA     	   32487300051/GRACIMAR BRASIL GUERRA     	   12455426050-JUCILENE MOREIRA CRUZ  
 @     98753936060'MARIVALDA KANAMARY          22222222222+Juscelino Kubitschek    	 
```

> Perceba agora que as **três versões** do registro `1001` coexistem fisicamente na página de 8 KB: a original (`MARIVALDA KANAMARY` em `00001f60`), a primeira atualização (`MARI K.` em `00001d98`) e a segunda atualização (`MARIVALDA DE ALCÂNTARA...` em `00001cc8`). As versões antigas (tuplas mortas) permanecem no arquivo até que uma rotina de limpeza (`VACUUM`) libere o espaço para reutilização.


## Parabéns!

Você concluiu com sucesso o laboratório de armazenamento em linha com PostgreSQL! 🎉

Espero que este exercício tenha proporcionado uma compreensão prática sobre o funcionamento do modelo de armazenamento baseado em linha e como o PostgreSQL gerencia as páginas e versões de registros no disco.

## Checkpoint (Flush)

Caso necessário, é possível forçar imediatamente a descarga das páginas sujas da memória (`shared_buffers`) para o arquivo de dados em disco executando um `CHECKPOINT`:
```
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

4. **Row-Oriented vs. Column-Oriented Storage** (conceito geral)
   https://en.wikipedia.org/wiki/Column-oriented_DBMS

5. **Designing Data-Intensive Applications** — Martin Kleppmann
   Capítulo 3: "Storage and Retrieval" — cobre modelos de armazenamento em linha vs. coluna.

## Apêndice: Execução Local

Fora do horário de aula, é possível executar o mesmo ambiente em qualquer máquina (ou provedor de nuvem) com Docker disponível. A imagem é publicada para `linux/amd64` e `linux/arm64` (inclusive Macs com Apple Silicon).

Faça o clone deste repositório e inicialize o container com o `compose.yaml` disponível na raiz do projeto:
```
git clone https://github.com/infobarbosa/postgres-rowstore-demo.git
cd postgres-rowstore-demo
docker compose up -d

```

Ou, sem clonar o repositório:
```
docker run -d --name postgres-lab -p 8080:8080 ghcr.io/infobarbosa/postgres-lab-docker-image:latest

```

Acesse `http://localhost:8080` no navegador e siga o tutorial a partir do [Passo 3](#passo-3-acessar-a-ide).

Para encerrar e remover o container (os dados do PostgreSQL serão descartados):
```
docker compose down

```
ou
```
docker rm -f postgres-lab

```
