

#Propagação de erros

Defina um tipo erro para fazer a propagação do erro através do resultado da função.

```cpp
//Result.h

typedef enum 
{
  RESULT_OK,
  RESULT_FAIL,
  RESULT_OUT_OF_MEM,
} Result;
```
Adicione apenas os erros que você está usando. 
Não crie erros para o futuro e remova erros não usados. 


(Uma ferramenta de análise estática deveria procurar por todas as comparações (if/switches) de variáveis do tipo Result com cada tipo de erro.
Se o tipo de erro não for usado deveria apresentar um warning. A ferramenta em modo "lib" deve procurar por escritas. A diferenças entre uma "lib" é que o uso de um código de erro pode ser apenas um escrita pois a leitura pode não estar visível.)
Qualquer enum poderia usar estas regras.

Logo que precisar, crie uma função para converter código de erro em texto.



#Objetos em C


##Declaração

```cpp
typedef struct 
{
  ...
} T;
```

##Inicialização estática
A inicialização de objetos pode ser feito em uma linha através de uma atribuição.

```cpp
T obj = T_INIT;
```

##Funções comuns

###T_Init

```
Result T_Init(T* p);
```

Inicializa o objeto p.
O chamador deve garantir que p não é nulo.
Uma característica desta função é que p é um parâmetro out não inicializado. Ou seja, a princípio p contém lixo.
A implementação deve garantir que nenhum leak ocorre em caso de falha.
Caso a Init não tenha sucesso, nenhuma outra função que use T pode ser usada, incluindo a T_Destroy.
Se Init teve sucesso  o _Destroy deve ser sempre chamado.
Nem toda Init falha. No entanto definir um resultado para init auxilia na homogenizacao do conceito de Init / Destroy através da criação do bloco if.
O destroy neste caso sempre é chamado precedendo um } o que torna fácil acompanhar o par init destroy através do alinhamento das chaves.


Isto automaticamente cria o seguinte padrão:
```cpp
int main()
{
   T t;
   Result result = T_Init(&t);
   if (result == RESULT_OK)
   {
     T_Destroy(&t);
   }
}
```

##T_Destroy
```
void T_Destroy(T* p);
````
O destroy é usado pelo chamador para indicar o término do uso do objeto p. 

O chamador deve garantir que p não é nulo.
Depois do destroy p não pode mais ser usado.
O implementador usa o destroy para liberar qualquer recurso e destruir os objetos filhos.
O Destroy não é uma função obrigatória. Tem todos os objetos precisam liberar recursos na destruição.
No entando a existência da função Destroy obriga a sua chamada após a Init.

###T_Create
```cpp
Result T_Create(T**pp);
```
Cria o objeto T fora da pilha e depois inicializa o objeto.
Caso a função falhe nenhum leak é criado.
Uma função Init sempre existe, mas uma create é opcional. Quando a função Create existir, a Init também pode virar "private" caso o objeto nunca seja alocado na pilha e somente fora dela.

Crie uma função Malloc para funcionar com o allocador padrão. E da mesma forma Free para liberar a memória do alocador padrão.
Criar e usar somente estas funções auxilia na prevenção de memory leaks.

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

void T_Delete(T* p)
{
   if (p != NULL)
   {
     T_Destroy(p);
     Free(p);
   }
}

Destroi o objeto p e devolve a memória para o alocador.
Obrigatoriamente, se p for nulo a função não tem efeito.
Caso exista uma função Create haverá uma Delete.

###T_Reset

```cpp
Result T_Reset(T* p)
{
     T_Destroy(p);
     return T_Init(p);
}
```

Reset é uma utilitária, destroi o objeto e volta para o estado correspondente ao init.


###T_Swap
```cpp
void T_Swap(T* p1, T* p2);
{
    T temp = *p2;
    *p2 = *p1; 
    *p1 = temp;
}
```
A swap copia o estado do objeto p1 para o objeto p2 e vice versa.
p1 e p2 estão inicializados.

###T_Clear
A função clear não destroy o objeto. Ela coloca o objeto em um estado válido vazio que não corresponde ao Reset.
O Reset coloco no estado identico ao Init. A clear ,por exemplo, pode zerar uma string sem desalocar memória. 


##T_InitCopy
```cpp
Result T_InitCopy(T* p1, T* p2);
{
    
}
```
Copia, duplica, p2 incluindo todos as partes para p1 que não está inicializado.

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



##T_PtrMoveTo
```cpp
void T_PtrMoveTo(T** p1, T** p2);

```
TODO 

##Funções

Todo parâmetro do tipo ponteiro é por padrão não-nulo e input.

```
void F(T* p)
{
  printf("%d", p->i);
}
```

Qualquer comparação por nulo, torna o ponteiro opcional.


Qualquer atribuição torna o ponteiro out.
Qualquer uso antes da atribuição torna o ponteiro in-out.


```
void Get(int *p)
{
   *p = 1; //out
}
```
Por padrão os objetos out estão inicializados. A init é a exceção.


##Custódia transferida de fora para dentro de uma função.

A transferência de custódia de um objeto para o parâmetro de uma função pode ser de duas formas.
Incondicional, aonde a custódia é transferida independente do sucesso da função. Ou condicionado ao sucesso.

```
//Aqui a transferência é incondicional
Result Array_Add(List* pArray, T * pItem)
{
  Result result = Array_Reserve(pArray, pArray->size + 1);
  if (result == RESULT_OK)
  {
  
  }
  T_Delete(pTOwner);
}
```

##Custódia transferida de dentro para fora da função.
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

Mantenha no header apenas as funções usadas por outros arquivos. Funções que são detalhes de implementação mantenha como static no arquivo c.

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





