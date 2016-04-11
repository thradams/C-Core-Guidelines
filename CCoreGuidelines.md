
#Introdução
O objetivo deste manual é descrever padrões e práticas para se criar softwaer em C.
Existem muitos estilos diferentes seja identacacao nomes padroes de programacao.
O estilo aqui, não prendente se colocar como a melhor maneira de se fazer software em C, 
porém como uma maneira real que funciona para qualquer tamanho de projeto e que diminui a existencia de bugs,
de memory leaks.


#Propagação de erros

Em C, umas das melhores maneiras, senão a melhor, de se propagar erros
é através do retorno das funções.

 * Use enum para definir um tipo de erro e para garantir que cada valor é único
 * Coloque em um header só para isso
 * Adicione apenas os erros que você está usando 
 * Não crie erros para o futuro e remova erros não usados 
 * Se você estiver usando um protocolo coloque as constantes numéricas
 * Logo que precisar, crie uma função que converta o valor para texto. Crie o .c equivalente com a implementação.
 * Se estiver criando uma lib inclua um sufixo no tipo e valores
 
```cpp
//Result.h

typedef enum 
{
  RESULT_OK,
  RESULT_FAIL,
  RESULT_OUT_OF_MEM,
} Result;
```

Uma lib que foi criada para ser incluída em outro projeto, 
vai possuir seu próprio tipo e valores  de erros.
Isso ocorre porque não existe uma definição universal do erro,
embora os erros como falta de memória ou erros genéricos existirão 
em qualquer programa.

A maneira de se lidar com isso, é converter os erros da lib para erros locais do seu programa.
Como isso pode ser uma tarefa árdua pois nem sempre se tem uma lista clara de erros a outra alternativa
é propagar em conjunto 2 ou mais erros até chegar no códito que trata.

Exemplo.
Se você usar uma lib de sockets vai ter erros especificos. 
Você pode definir um valor RESULT_SOCKET_ERROR para indicar 
que a informação extra de erro (no caso o erro que vem da lib de sockets do SO) deve ser usada.

Quanto mais libs forem usadas mais complicado fica.
Em uma cadeia pequenas de chamadas pode ser uma boa solução.

```c
Result SentString(socket* s, const char* str, int sockerrors);

```
Quando a cadeia de erros é grande, ou seja, o ponto que o erro
é lançado é muito distante do ponto aonde o erro é tratado, e o número de erros possíveis é muito grande, 
o que eu recomendo é ignorar o número do erro complementar (algo que geralmente é possível)  e propagar a informação extra na forma de string.
A string, diferente do número de erro, aceita misturas de libs e propaga informação complementar indiferente da fonte. 
De certa forma a string é o propagador universal de informação extra.

Informações específicas, necessárias para tratar um caso podem ser individualmente incorporadas ao programa.

RESULT_SOCKET_ERROR_I_WILL_CHECK

Para a string de informação de erro extra, o ideal é criar um tipo.


#Objetos em C


##Declaração

```cpp
typedef struct 
{
  ...
} T;
```

Esta declação com typedef permite que o tipo seja usado sem a palabra struct na frente. 
Semelhante ao que acontece no C++.

Diferentemente do C++, em C qualquer coisa pode ser um objeto sem que seja preciso criar wrappers.

```c
typedef int MyInt;
```
Neste caso acabo de criar a classe MyInt que funciona como um int mas que pode ser extendido.

```c
MyInt_IsOdd(MyInt i);
```

##Inicialização estática

A inicialização de objetos pode ser feito em uma linha através de uma atribuição.
T_INIT é uma macro que deve ser declarada junto do objeto.

Uma característica é que ela não falha. O ideal é usar este tipo de inicilização em objetos
aonde não seja preciso fazer limpeza ao fim do escopo.
Objetos deste tipo, podem precisar a inicialização por função também.
```cpp
T obj = T_INIT;
```

##Sintaxe e Semântica das funções comuns entre objetos
Os objetos possuem funcoes em comum indedenten do seu tipo.
Estas funcoes que possuem a mesma semantica podem ser indentificadas pelo nome e desta
forma o programador automaticamente conhece o seu funcionamento indendente do tipo. 
Para exemplicar a inportancia disso na verdade basta considerar a ausencia destas convencoes e
o tempo perdido na descoberta da semantica de criacao e limpeza de cada objeto.
Seguir estas funções significa entao acelerar muito o desenvolvido e diminuir a ocorrencia de bugs.
 
##Init / Destroy

###T_Init

```cpp
Result T_Init(T* p);
```

Inicializa o objeto p.
O parâmetro p é não nulo out e não inicializado.

A implementação deve garantir que nenhum leak ocorre em caso de falha.
Caso a Init não tenha sucesso, nenhuma outra função que use T pode ser usada, 
incluindo a T_Destroy.
Se Init teve sucesso  o _Destroy (se existir) deve ser sempre chamado.
Nem toda Init falha. No entanto definir um resultado para init auxilia na 
homogenizacao do conceito de Init / Destroy através da criação do bloco if. 
Outra vantagem é caso o objeto seja modificado e passe e dar erro no Init, você nao precisa
reescrever os chamadores do objetos, pois eles estão preparados.

Isto automaticamente cria o seguinte padrão:

```cpp
int main()
{
   T t;
   Result result = T_Init(&t);
   if (result == RESULT_OK)
   {
     //uso de t
     T_Destroy(&t);
   }
}
```

##T_Destroy
```cpp
void T_Destroy(T* p);
````
O destroy é usado pelo chamador para indicar o término do uso do objeto p. 
O chamador deve garantir que p não é nulo.
Depois do destroy p não pode mais ser usado.
O implementador usa o destroy para liberar qualquer recurso e destruir os objetos filhos.
O Destroy não é uma função obrigatória. 
Nem todos os objetos precisam liberar recursos na destruição.
No entando a existência da função Destroy obriga a sua chamada após a Init. Isso garante um conceito homegênio que evita bugs.

##Create / Delete

###T_Create
```cpp
Result T_Create(T**pp);
```
Cria o objeto T fora da pilha e inicializa ele com Init. Caso o Init falhe, ele destroy o objeto e a função não tem efeito colateral.
Uma função Init sempre existe, mas uma create é opcional. Ela é usada quando o tipo de objeto vai ser criado fora da pilha.
Algumas vezes um determinado tipo somente é criado fora da pilha, neste caso deixe a função Init/Destroy apenas como detalhe de implementação e publica no header apenas a Create e Delete.
Se o objeto for hora criado no heap ora na pilha, mantenha no header as 4 opções. Init/Destroy e Create/Delete.
somente fora dela. Para um objeto que nunca é criado na pilha considere também deixá-lo declarado no arquivo .c. 
Com isso você maior encapsulamento sem custo algum. 

Na implementação da Create, caso ela use o heap, não utilie o malloc diretante. Faça seu próprio Malloc e Free.
Isso permite que você implemente um detector de memory leaks ou mude a estratégia do alocador global sem mexer mais no seus tipos.

Aqui está a função completa para um tipo T.

```cpp
Result T_Create(T**pp)
{
 Result result = RESULT_OUT_OF_MEM;
 
  T* p = (T*) Malloc(sizeof(T) * 1);
  if (p != NULL)
  {
     result = T_Init(p);
     if (result == RESULT_OK)
     {
        *pp = p;
     }
     else
     {
       Free(p);
     }
  }
  return result;
}
```

###T_Delete
```cpp
void T_Delete(T* p)
{
   if (p != NULL)
   {
     T_Destroy(p);
     Free(p);
   }
}
```

Destroi o objeto p e devolve a memória para o alocador.
Obrigatoriamente, se p for nulo a função não tem efeito. 
Isso é usado para simplificar o número de caminhos em algumas funções. Caso você tenha um caso crítico de performance crie a UnsafeDelete que não testa por nulo.
O padrão da delete, segue desta forma o mesmo padrão do free do C e delete do C++. Ou seja, é "safe" testa por null.
Caso exista uma função Create haverá uma Delete.

###T_Reset

Função completa
```cpp
Result T_Reset(T* p)
{
     T_Destroy(p);
     return T_Init(p);
}
```

Reset é uma utilitária, destroi o objeto e volta para o estado correspondente ao init.
Se o Reset falhar o objeto não poderia ser mais usado, ou seja, a "reciclagem" falhou.

###T_Swap
```cpp
void T_Swap(T* p1, T* p2);
{
    T temp = *p2;
    *p2 = *p1; 
    *p1 = temp;
}
```
Em raros casos esta implementação não é a definitiva.
Quando o objeto possui ponteiros para si mesmo uma cópia simples de memória não é suficiente.
A swap copia o estado do objeto p1 para o objeto p2 e vice versa.
p1 e p2 estão inicializados.

###T_Clear
A função clear não destroy o objeto.
Ela coloca o objeto em um estado válido vazio que não corresponde obrigatoriamente ao Reset.
A clear ,por exemplo, pode zerar uma string sem desalocar sua memória. 


##T_InitCopy
```cpp
Result T_InitCopy(T* p1, T* p2);
{
    
}
```
Inicializa p1 com uma cópia de p2.

##T_Copy
```cpp
Result T_Copy(T* p1, T* p2);
{
    T temp;
    Result result = InitCopy(&temp, p2);
    if (result == RESULT_OK)
    {
      //este é o move perigoso
      //aquele que o objeto movido (temp) não pode mais ser usado nem destruído
      //em pequenos contextos como este ele fica claro e eficiente. 
      T_Destroy(p1);
      *p1 = temp;    
    }
    return result;
}
```
Copia p2 para p1. p1 e p2 devem estar inicializados.
Raramente esta não é a implemetação final.


##T_MoveTo (safe)
```cpp
void T_MoveTo(T* p1, T* p2);
{
      T_Destroy(p2);
      *p2 = *p1;
      T_Init(p1);    
}
```
p1 e p2 estao inicializados. p1 é movido para p2 e depois fica no estado init que deve ser destruido.
Este move então é util para fazer caminhos sem returns soltos aonde sempre é passado pelo destructor do objeto. 
No caso de sucesso o objeto estara movido e em caso de erro ele fara a destruicao normal.

Podemos ter o UnsafeMoveTo ou DismissMoveTo aonde o objeto não pode nem ser destruído.

##T_PtrMoveTo
```cpp
void T_PtrMoveTo(T** p1, T** p2);

```
TODO 
##InitMove
Move o objeto de entrada para o objeto que está sendo inicializado. O objeto de entrada
fica em um estado que pode ser destruido.
Ele é movido condicionamente ao sucesso do init.
Na verdade eh melhor usar um caso simples. aonde o move coloca o obeto no estado do init.
A impleemntacao eh mais simles  e deveria ser o default.
Entao um move otimizado eh que deveria ser diferenciado e so vale a pena em casos muitos especiais.
Poderia ser chamar InitMoveDismiss
O move cia uma situacao especial para otimizar e evitar a limpeza de algusn items.

##Convenções

Todo parâmetro do tipo ponteiro é por padrão não-nulo.

```
void F(T* p)
{
  printf("%d", p->i);
}
```

Qualquer comparação por nulo, torna o ponteiro opcional.
Qualquer uso do ponteiro torna ele IN.
Qualquer atribuição torna o ponteiro out.
Qualquer uso antes da atribuição torna o ponteiro in-out.


```
void Get(int *p)
{
   *p = 1; //out
}
```
Por padrão os objetos out estão inicializados. A init é a exceção.
Objetos out não inicializados são aqueles que nascem na função que é só o caso da Init.


##Custódia transferida de fora para dentro de uma função.

Casos
 * Mover um objeto inicializado (in out)
 * Mover um objeto sob a custodia de um ponteiro 
 
A operacao de mover um objeto nao precisa ter nenhuma diferenca em relacao a ser um objeto in out.
Por exemplo, um objeto strint in out que entrou com um valor e saiu vazio. Do ponto de vista do objeto ele apenas foi modificado.
A diferenca entre in out e movido ocorre apenas se quisermos otimizar o move de forma que o ojbeto nao possa mais ser utilizado depois.
Esta otmizcao na pratica ser ser a atruibuicao de algumas membros do ojeto. Entao para o conceito geral eh melhor deixar
o move especial como uma otimizacao.


A transferência de custódia de um objeto para o parâmetro de uma função pode ser de duas formas.
Incondicional, aonde a custódia é transferida independente do sucesso da função. 
Ou condicionado ao sucesso.

```c
//Aqui a transferência é incondicional
Result Array_Add(List* pArray, T * pItem)
{
  Result result = Array_Reserve(pArray, pArray->size + 1);
  if (result == RESULT_OK)
  {
    T_InitMove(&pArray->data[pArray->size], item);
    
  }
  else
  {
      T_Destroy(pItem);
  }
}

void F1()
{
 T item;
 Result result = Array_Add(&array, &item); //transferido incondicionamente
 if (result == RESULT_OK)
 {
     
 }
}
```

```c
//Aqui a transferência é condicional
Result Array_Add(List* pArray, T * pItem)
{
  Result result = Array_Reserve(pArray, pArray->size + 1);
  if (result == RESULT_OK)
  {
    T_InitMove(&pArray->data[pArray->size], item);
    pArray->size++;
  }
}


void F2()
{
 T item;
 Result result = Array_Add(&array, &item); //transferido condificonamente
 if (result == RESULT_OK)
 {
     
 }
 //nao eh seguro usar item aqui pode ter sido movido
 T_Destroy(&item); //pode ou ter sido movido
}


```
Trasferencia de objetos no heap

```c
Result Array_Add(List* pArray, T ** pItem)
{
  //ponteiro p ponteiro com custodia nao nulo entrou na funcao
  //e pode ser movido
  Result result = Array_Reserve(pArray, pArray->size + 1);
  if (result == RESULT_OK)
  {
    //o conteudo de item passou a ser do array
    //o ownership do conteudo foi movido
    pArray->data[pArray->size] = *item;
    *item = NULL;
    //T_PtrInit
    //T_PtrInitMove
    pArray->size++;
  }
}


void F2()
{
 T* item;
 Result result = T_Create(&item);
 if (result == RESULT_OK)
 {
   result = Array_Add(&array, &item); //transferido condificonamente
   if (result == RESULT_OK)
   {
     
   }
   //nao eh seguro usar item aqui pode ter sido movido
   T_Delete(item);
 } 
}


```

Trasferencia de objetos no heap incondicional

```c
Result Array_Add(List* pArray, T * item)
{
  //item inicializado com custodia entrou na funcao
  //e poed ser movido
  Result result = Array_Reserve(pArray, pArray->size + 1);
  if (result == RESULT_OK)
  {
    //o conteudo de item passou a ser do array
    //o ownership do conteudo foi movido
    pArray->data[pArray->size] = *item;
    *item = NULL;
    //T_PtrInit
    //T_PtrInitMove
    pArray->size++;
  }
  //nao eh seguro usar item aqui pode ter sido movido
  T_Delete(item);
}


void F2()
{
 T* item;
 Result result = T_Create(&item);
 if (result == RESULT_OK)
 {
   result = Array_Add(&array, item); //transferido incondificonamente *
   if (result == RESULT_OK)
   {
     
   }
   //aqui nao tem delete
   
   //poderia deixar o delete e mover ** incondicionamente
   //mas dai o delete ja se sabe que eh descnecessario.
 } 
}


```
A conclusao que o melhor estilo eh mover condificonamente.
Move ponteiros com ponteiro de ponteiro.

Também dá para fazer transferencia de custodia com swap.
O conteudo de um item passa para o outro e vice versa. Isso pode ser usado
para funcoes do tipo substituicao de um item. Ao inves de um Add seria um "SetAt" ou
"Replace"ou "Change". Um move pode deixar um objeto em um estado incosistente, ja o swap deixa smepre
em um estado que foi modificado mas consistnte.

A trasnferecia de custodia precisa ser documentada pois o chamador tem que estar ciente
que o objeto pode ficar em um estado invalido e nao pode mais ser usado apenas destruido.
Poderia ter um macro MAY_MOVE(T). De certa forma é melhor nao tentar averiguar se foi movido ou nao.
pois a intencao era mover, e o uso do objeto eh apenas para faciliar a destruicao de forma homegenia.
Quando a transferencia ocorre por ponteiro obrigatoriamente o ponteiro de entrada tinha custodia.


```c
 T* item;
 Result result = T_Create(&item);
 if (result == RESULT_OK)
 {
     
   result = Array_Add(&array, MOVE(item));
   result = Array_Add(&array, &item /*move*/);
   if (result == RESULT_OK)
   {
     
   }
   T_Delete(item);
 }
```

##Custódia transferida de dentro para fora da função.

Casos

* Ponteiro out não inicalizado para custodia
* Pointeiro in out com custodia.
* Ponteiro sem custodia (poderia ser in out ou out ou nao inializado)
* Objeto não inicializado. (out)
* Objeto inicializado (in out)
 
Geralmente este caso ocorre na producao de objetos. Objetos criados sao enviados
para fora. Desta forma o valores de entrada que irao receber os objetos criados
podem ser nao inicializados.
É possivel fazer um swap e neste caso o objeto de entrada deve estar inicialiozado
Ou daria para fazer um InitMove considerando que ele nao esta inicializdo
O problema eh que ele seria inicializado dependendo do sucsso da funcao. E neste
caso do caminho do destroy seria assim.
Outra situacao eh a mudanca de um objeto. Neste caso é um in out e não é preciso entender como
custodia modificada. APenas o pbjeto foi alterado.
Ja no caso de ponteiro eh diferente. Pois um ponteiro modificado é preciso saber se eh um ponteiro 
que tem custodia.

Versao out nao inicializado
```c

T item1;
T item2;
//neste caso sao outs nao inicialiazados.

result = X_Create(&item, &item2);
if (result == RESULT_OK)
{
  //tudo ou nada
  T_Destroy(&item1);   
  T_Destroy(&item2);   
}


```

Versao out inicializado. (Swap)
```c
//este exemplo estaria protegido contra um nao "tudo ou nada"
T item1;
result = T_Init(&item);
if (result == RESULT_OK)
{
  T item2;
  result = T_Init(item2);
  if (result == RESULT_OK)
  {
     //em outras palavras aqui item1 e item sao in out
     result = X_Create(&item1, &item2);
     if (result == RESULT_OK) 
     {
        
     }
     T_DestroY(&item2);      
  }
  T_DestroY(&item1);      
}

```
No caso de ponteiros nao custa nada inicializar para null. Neste caso a funcao que entrega a custodia para fora
pode ter um assert pelo ponteiro nulo.
A vantagem da entrada inicializada eh que a mesma funcao pode ser usada para criacao ou mudanca.


Neste caso a custódia é sempre condicionada ao sucesso da função e nunca é parcial.

Todo retorno const char* de função é considerado como não passando a custódia;

```
const char* T_GetName(T *p)
{
   return p->name;
}
```

Strings em funções são passadas como const char*.
char* nuna deve significar número. Para isso use byte ou uint8_t.

#Distinção de ponteiros com custódia

O tipo de um ponteiro com custodia é o mesmo que o sem. Ou seja, esta informacao nao esta
no ponteiro.
O que e possivel fazer é um typedef para indicar que existe custodia ou um sufixo no nome
da variavel para indicar.

Um ponteiro pode ser transformado em uma classe no C e ter a mesma semantica de 
um objeto comum.

Um caso muito comum e que pode gerar muita confusao sao as strings do C.

#Custódia de Strings

Utilize a classe StringC para representar a custódia de uma string do C.

```
typedef char* StringC;
```

```
struct X
{
  StringC name;
};

Result X_Init(X * px)
{
   Result result = StringC_Init(&px->name, "");
   return result;
}

X_Destroy(X* px)
{
  StringC_Destroy(&px->name);
}

Result X_SetName(X* px, const char * name)
{
  return StringC_Change(&px->name, name);
}

const char* X_GetName(X* pX)
{
   return px->name;
}

```

Não use StringC como parâmetro de funções.

```
void X_SetNameMove(X* px, StringC* s)
{
    StringC_MoveFromTo(s, &px->name);
}

void X_SetName(const char* text)
{
  StringC_Attach(px->name, text);
}

void X_GetName(X* p, char** ppText)
{
  *ppText = StringC_Detach(p->name);
}
```

Quando const char* for membro de struct, por padrão ele não tem custódia.
Ou seja, o escopo de vida de Info deve ser menos que name.
```
typedef struct
{
  const char* name;
} Info;

void F(Info *p)
{
  ..
  p->name ..
  ..
}


```
##Arquivos
Utilize um arquico .h e outro .c para declarar e implementar uma classe.

##Alteração do estado de uma classe.

Toda alteração de estado de uma classe ,por padrão, é feita por funções declaradas no .h ou .c da classe.
Não seguir esta regra é extremamente perigoso já que o C não possui encapsulamento.

##Visualização do estado de uma classe
Acesse diretamente os dados de uma classe apenas para leitura e apenas dos membros da classe que você considera oficialmente parte da interface.
Para todos os outros user "Getters"

##Encapsulamento

Mantenha no header apenas as funções usadas por outros arquivos.
Funções que são detalhes de implementação mantenha como static no arquivo c.

##Encapsulamento de objetos criados no heap
Para objetos criados sempre no heap, esconda a função Init e use ponteiro opacos para esconder os detalhes de implementação.

##Encapsulamento de objetos criados na pilha
Utilize "Getters" e "Setters" para qualquer operação feita no objeto. Você pode definir uma macro para a inicialização estática.


#Polimorfismo no C



#Asserts

Não utilize assert.h diretamente.
Use ASSERT para permitir outras configurações.


#Config.h
Include primeiramente o arquivo Config.h.
Este este arquivo para preparar o ambiente de compilação de acordo com a plataforma.

#Closures em C

#Custódia em calbacks e void*


#Private
Existem libs que simulam o private mudando os nomes dos membros de uma struct em release.


