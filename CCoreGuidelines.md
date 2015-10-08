

#Propagação de erros

Defina um tipo erro para fazer a propagação do erro através do resultado da função.

```
typedef enum 
{
  RESULT_OK,
  RESULT_FAIL,
}Result;
```
Adicione apenas os erros que você está usando. 
Não crie erros para o futuro e remova erros não usados.
Logo que precisar, crie uma função para converter código de erro em texto.



#Objetos em C

Use uma semântica consistente.

T_Init
T_Destroy
T_Delete
T_Create
T_Copy
T_Swap

###T_Init

```
Result X_Init(X* px);
```

Inicializa o objeto X.  
Caso tenha sucesso na inicialização a finalização (X_Destroy) deve ser chamada. 
Em caso de erro, a instância não pode ser usada e nem destruída. 
Em caso de erro nenhum leak ou efeito colateral vai ser criado.
px nunca é nulo garantido pelo chamador.

##T_Destroy
```
void T_Destroy(X* px);
````

Destroi o objeto px. Se px for nulo não faz nada.
Depois da chamada desta função px não pode mais ser usado com exceção da função X_Init.

###T_Create
```
T*  T_Create();
Result T_CreateEx(T**pp);
```
Utiliza um alocador para criar o objeto T no heap e depois inicializa o objeto. Caso a função falhe nenhum leak é criado.

Implementação típica.
```
T* T_Create()
{
  T* p = (T*) Malloc(sizeof(T) * 1);
  if (p != NULL)
  {
     Result result = T_Init(p);
     if (result != RESULT_OK)
     {
       Free(p);
       p = NULL;
     }
  }
  return p;
}

Result T_CreateEx(T**pp)
{
  T* p = (T*) Malloc(sizeof(T) * 1);
  if (p != NULL)
  {
     Result result = T_Init(p);
     if (result == RESULT_OK)
     {
        *pp = p;
     }
     else
     {
       Free(p);
     }
  }
  else
  {
     result = RESULT_OUT_OF_MEM;
  }
  return result;
}
```
Utilize um alocador próprio (Malloc/Free), desta forma você pode verificar leaks e testar a quantidade de memória necessária para seu programa.

##T_Delete

```
void T_Delete(X* px)
```

Destrói o objeto px e devolve a memória para o alocador.
Obrigatoriamente, se px for nulo a função não tem efeito.

Implementação típica

```
void X_Delete(X* px)
{
   if (px != NULL)
   {
     X_Destroy(px);
     free(px);
   }
}
```
##T_Reset

Reset, destroi o objeto e volta para o estado correspondente ao init.

Implementação típica
```
void X_Reset(X* px)
{
     X_Destroy(px);
     X_Init(px);
}
```

##Funções


Todo parâmetro do tipo ponteiro é por padrão não-nulo e input.

Exemplo:

```
void F(X* px)
{
  //Nada declarado então px é IN não nulo.
}
```

Parâmetros do tipo ponteiro out deve ser comentados no momento da atribuição. 
Por padrão, ele é considerado não nulo.
```
void Get(int *p)
{
   *p = 1; //out
}
```
Caso o parâmetro out seja opcional, comentar no if.
```
void Get(int *pOpt)
{
    if (pOpt != NULL) //optional
    {
       *pOup = 1; //out
    }
}
```

Parâmetros IN opcionais devem ser comentados ou ter sufixo Opt.

```
void F(X* pxOpt)
{
  if (pxOpt != NULL)
  {
    ... usar pxOpt
  }
}
```

Todos os parâmetros ponteiros são considerados não donos do conteúdo a não ser que informe o contrário.

Caso a função receba um ponteiro da qual é dona ela deve informar no momento da destruição ou no momento da trasferência de ownership.

##Custódia transferida de fora para dentro de uma função.

A transferência de custódia de um objeto para o parâmetro de uma função pode ser de duas formas.
Incondicional, aonde a custódia é transferida independente do sucesso da função. Ou condicionado ao sucesso.
Esta informação deve ser obrigatoriamente informada na função e comentada no chamador.

```
Result Array_Add(List* pArray, T * pItem)
{
  Result result = Array_Reserve(pArray, pArray->size + 1);
  if (result == RESULT_OK)
  {
  
  }
  T_Delete(pTOwner);
}
```


Para casos exceptionais de performance pode-se ignorar a atruibição para nulo.

##Custódia transferida de dentro para fora da função.
Neste caso a custódia é sempre condicionada ao sucesso da função e nunca é parcial.


```
//Atenção: Chamador deve ignorar px após chamada desta função
//Com sucesso ou falha ela ficou responsável por px.
Result F(X* px)
{
  X_Delete(px); //owner e chamador deve ignorar px após a chamada desta função.
}
```

Qualquer mudança de ownership deve ser comentada. A não existencia de comentários indica que é apenas uma referência fraca.

```
void F(X** ppx)
{
  X* px = *ppx;
}
```

Todo retorno const char* de função é considerado como não passando a custódia;

```
const char* T_GetName(T *p)
{
   return p->name;
}
```
```
const char* GetErrorMessage(Error e)
{
   switch (e)
   {
      case E1: return "E1";
   }
   return "";
}
```

Transferencia de custódia de ponteiros que saem da função.
A custódia só será transferida caso a função tenha sucesso.
```
Result T_Clone(T **ppT)
{
   *ppT = pnew; //out
}
````

Em poucos casos.
```
X* X_Create()
{
   return pnew; //out
}
````

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

##Nomes de funções de classe
Use o nome da classe T_ seguido do nome da função. Para funções estáticas declaradas no .c, não é preciso seguir esta padronização.

#Polimorfismo no C



#Asserts

Não utilize assert.h diretamente.
Use ASSERT para permitir outras configurações.


#Config.h
Include primeiramente o arquivo Config.h.
Este este arquivo para preparar o ambiente de compilação de acordo com a plataforma.

#Visual C++
inline no C








