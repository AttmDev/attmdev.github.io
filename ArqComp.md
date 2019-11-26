<font size="4">

# Arquitetura de computadores
## Resumo

###   Itens:

1. **Arquitetura vetorial**
2. **Extensões *SIMD* para multimídia**
3. **Unidades de processamento gráfico (GPUs)**
4. **Paralelismo em nível de loop**

-----------------------------


## Arquitetura vetorial

As arquiteturas vetoriais realizam a mesma operação sobre múltiplos elementos. Um computador  com arquitetura vetorial carrega múltiplos valores para um registrador vetorial, o processador opera uma única instrução para todos os dados e armazena os dados modificados na Memória principal.

>O livro utiliza a arquitetura **VMIPS** para introduzir o conceito de arquitetura vetorial.

Essa arquitetura possui os seguintes componentes principais:

- **Registradores vetoriais:** Cada registrador vetorial é um banco de memória de tamanho fixo mantendo um vetor. O VMIPS possui 8 registradores com 64 elementos cada.
- **Registradores escalares:** Registradores escalares podem fornecer dados de entrada para as unidades funcionais vetoriais e podem calcular endereços para carregamento e escrita dos dados.
- **Unidade carregamento-armazenamento vetorial:** Pode carregar dados da memória principal para os registradores vetoriais e escalares tanto quanto escrever dados desses registradores na memória principal. Os carregamentos e armazenamentos são *pipelined*, o que permite transferência de palavras com banda de uma palavra por ciclo, após a latência inicial
- **Unidades funcionais vetoriais:** O VMIPS possui 5 unidades funcionais, sendo uma de controle para detectar riscos e as outras para calculo em ponto flutuante e em inteiros. Cada unidade é totalmente *pipelined*, podendo iniciar uma nova operação a cada ciclo de clock.


>As instruções do VMIPS utilizam os mesmos nomes da arquitetura MIPS, com adição das letras VV, no caso de operações entre 2 vetores e VS no caso de operação entre um vetor e um escalar. Assim, o código ***ADDVV.D V1,V2,V3*** adiciona os elementos de V2 e V3 e armazena-os em V1 e ***ADDVS.D V1,V2,F0*** soma o valor de F0 a cada valor de V2 e armazena em V1.
>
>Além disso para carregar e armazenar o registrador V1, a partir do endereço da memória R1, utiliza-se os comandos ***LV V1,R1*** e ***SV R1,V1***

#### Exemplo de funcionamento

Para entender melhor um processador vetorial, utiliza-se dois códigos um loop vetorial implementados no MIPS e no VMIPS. Esses códigos são chamados de ***SAXPY*** e ***DAXPY*** e realizam o seguinte calculo:
$$
Y = a * X + Y
$$
onde **X** e **Y** são vetores, e **a** é um escalar. Esse loop é uma parte do benchmark Linpack. Nesse caso vamos considerar que o tamanho do registrador vetorial(64) é igual ao tamanho da operação que realizaremos.

O código MIPS para o loop é esse:
```
				L.D				F0,a				;carrega escalar a
				DADDIU		R4,Rx,#512	;último endereço a carregar
Loop:		L.D				F2,0(Rx)		;carrega X[i]
				MUL.D			F2,F2,F0		;a * X[i]
				L.D				F4,0(Ry)		;carrega Y[i]
				ADD.D			F4,F4,F2		;(a * X[i]) + Y[i]
				S.D				F4,9(Ry)		;armazena em Y[i]
				DADDIU		Rx,Rx,#8		;incrementa índice de X
				DADDIU		Ry,Ry,#8		;incrementa índice de Y
				DSUBU			R20,R4,Rx		;calcula limite
				BNEZ			R20,loop		;verifica se terminou
```
Agora em VMIPS, o DAXPY fica assim:
```
				L.D				F0,a				;carrega escalar a
				LV				V1,Rx				;carrega vetor X
				MULVS.D		V2,V1,F0		;multiplicação escalar vetorial
				LV				V3, Ry			;carrega vetor Y
				ADDVV.D		V4,V2,V3		;soma
				SV				V4,Ry				;armazena o resultado
```
Uma das maiores vantagens do VMIPS em relação ao MIPS é que enquanto o primeiro utiliza somente 6 instruções o MIPS utliza quase 600.


#### Tempo de execução vetorial

O Tempo de execução de um programa vetorial depende de 3 fatores:

1. Tamanho dos vetores
2. Conflitos estruturais entre as operações
3. Dependencias de dados

O tempo de uma única instrução vetorial pode ser obtido a partir do tamanho do vetor e a *taxa de iniciação*. Nossa implementação do VMIPS tem apenas uma pista de pipelinec com taxa de iniciação	de um elemento por ciclo, de forma que o tempo de execução para uma instrução vetorial é o tamanho do vetor.

O conjunto de instruções que podem ser executadas no mesmo período de clock são chamadas de ***convey***. Para que as instruções sejam executadas em convey, elas não podem ter nenhum conflito estrutural ou de dados. Para que não haja problema com riscos de dependencia ***RAW*** usa-se uma tecnica chamada ***chaining*** que permite que uma operação com dependencia receba o resultado assim que ele fique disponível.

Para conseguir o tempo de execução do programa, utilizamos uma medida de temporização que estima o tempo de um convey chamada de *chime*. Dessa forma um programa vetorial que consistem em *m* conveys será executado em *m* chimes, e no caso de um tamanho vetorial *n* o programa sera executado em aproximadamente $m * n$ ciclos de clock. Utiliza-se a medida do chime ao invés de ciclos de clock por resultado para explicitar que alguns overhead estão sendo ignorados. Por exemplo no código em VMIPS a seguir:

```
				LV				V1,Rx				;carrega vetor X
				MULVS.D		V2,V1,F0		;multip vet por escalar
				LV				V3, Ry			;carrega vetor Y
				ADDVV.D		V4,V2,V3		;soma
				SV				V4,Ry				;armazena o resultado
```

Nesse caso temos o seguinte layout de conveys:

		1. LV			MULVS.D
		2. LV			ADDVV.D
		3. SV

Essa sequencia exige 3 conveys, ou seja usa 3 chimes. Além disso, como existem duas operações em ponto flutuante o número de ciclos por FLOP é de 1,5. E para vetores com 64 elementos necessitaria-se de 192 ciclos de relógio.

Uma fonte de overhead ignorada pelo modelo de chime é o tempo de início do vetor. Esse tempo vem da latência de pipeline da operação vetorial, determinado principalmente pela profundidade do pipeline da unidade funcional utilizada. A profundidade para cada unidade é:
Unidade  	    |   	  Ciclos
--------------|----------------
Float add     |     6 ciclos
Float mult    |     7 ciclos
Float divi    |    20 ciclos
Carregar vet  |    12 ciclos

Dados esses atrasos, existem alguns problemas que exigem otmizações que melhoram o desempenho ou diminuem o tamanho dos programas a ser executados nas arquiteturas vetoriais. São elas:

- Múltiplos elementos por ciclo de clock para executar um vetor mais rápido que um elemento por ciclo;
- Como a maioria dos programas não usa vetores de tamanho igual ao comprimento da arquitetura, utiliza-se um controlador de tamanho do registrador;
-  Se existe uma declaração de IF é necessário um controle para a execução das instruções;
- Como os processadores vetoriais precisam carregar e armazenar grandes quantidades de dados, eles se utilizam de bancos de memória para que não hajam esperas de dado;
- O processador pode ter que lidar com matrizes multidimensionais;
- O processador pode ter que lidar com matrizes dispersas.

###### Multiplas pistas de execução
Para acelerar as operações, o processador pode possuir varias combinações de unidades funcionais paralelas. Isso permite que uma operação vetorial ocorra bem mais rapido, pois partes da execução do programa acontecem ao mesmo tempo. Na estrutura do VMIPS existe a propriedade de que o n-ésimo elemento de um vetor só possa operar com o n-ésimo elemento de outro vetor, de forma que a unidade vetorial pode ser enxergada como uma rodovia. Entretanto, para que isso seja vantajoso, o programa e a arquitetura devem suportar vetores longos, caso contrário, as instruções são executadas muito rapidamente exigindo técnicas de paralelismo no nível de instrução.


##### Vetores de tamanho diferente do hardware

Para vetores de tamanho diferente do vetor da arquitetura, a solução é criar um registrador de tamanho de vetor, que controla o tamanho de qualquer operação vetorial e não pode ser maior que o tamanho dos registradores vetoriais.

Caso o o vetor tenha um tamanho desconhecido e/ou seja maior que o tamanho maximo do vetor, utiliza-se uma técnica chamada ***strip mining*** que é a geração de código de forma que cada operação vetorial seja feita para vetores menores ou iguais ao tamanho dos registradores. Para isso, cria-se um loop que trata de qualquer quantidade de iterações, que seja mútiplo do tamanho maximo, e um outro loop que trata as iterações restantes, que são menores que o tamanho do registrador. Na prática, os compiladores criam um único loop *strip-mined* que lida com as duas partes, alterando o tamanho em tempo de execução. Veja o exemplo do loop DAXPY em C:
```
int low = 0;
int VL = n % MVL; /*encontra a parte que sobra do loop*/
for (int j = 0; j <= (n/MVL); j++)/*loop externo*/
{
	for (int i = low; i < (low + VL); i++)/*Encontra a parte de tamanho impar*/
	{
		Y[i] = a * X[i] + Y[i];/*operação principal**/
	}
	low = low + VL; /*inicio do proximo vetor */
	VL = MVL; /* reinicia o tamanho com o comprimento maximo do vetor*/
}
```
O efeito do primeiro loop é bloquear o vetor em segmentos que são processados pelo loop interno. Dessa forma o tamanho do primeiro segmento é a parte do vetor menor que o tamanho do registrador(MVL) que é n % MVL e os segmentos restantes tem o tamanho do MVL.
</font>
