

#Propagação de erros

Defina um tipo erro para fazer a propagação do erro através do resultado da função.

```
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


(Uma ferramenta de análise estática deveria procurar por todas as comparações (if/switches) de variáveis do tipo Result com cada tipo de erro. Se o tipo de erro não for usado deveria apresentar um warning. A ferramenta em modo "lib" deve procurar por escritas. A diferenças entre uma "lib" é que o uso de um código de erro pode ser apenas um escrita pois a leitura pode não estar visível.)
Qualquer enum poderia usar estas regras.

Logo que precisar, crie uma função para converter código de erro em texto.



#Objetos em C


##Declaração

```
typedef struct 
{
  ...
} T;
```
##Funções comuns

###T_Init

```
Result T_Init(T* p);
```

Inicializa o objeto p.
O chamador deve garantir que p não é nulo. Uma característica desta função é que não interessa o estado de p. A princípio p contém lixo.
A implementação deve garantir que nenhum leak ocorre em caso de falha.
Caso a Init não tenha sucesso, nenhuma outra função que use T pode ser usada, incluindo a T_Destroy.
Se Init teve sucesso  o _Destroy deve ser sempre chamado.
Isto automaticamente cria o seguinte padrão:
```
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


###T_Create
```
Result T_Create(T**pp);
```
Utiliza um alocador para criar o objeto T no heap e depois inicializa o objeto.
Caso a função falhe nenhum leak é criado.
Uma função Init sempre existe, mas uma create é opcional para objetos que podem ou sempre são criados no heap.
Se o uso no código for sempre no heap mova a Init para o arquino .c.
A create assume um alocador padrão para o objeto já que nenhum alocador é passado como parâmetro. Ou seja, o mesmo objeto não pode ser usado por diferentes regras de alocação.

```
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
Utilize um alocador próprio (Malloc/Free), desta forma você pode verificar leaks e testar a quantidade de memória necessária para seu programa.

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

```
void T_Reset(T* p)
{
     T_Destroy(p);
     T_Init(p);
}
```

Reset é uma utilitária, destroi o objeto e volta para o estado correspondente ao init.


##Funções

Todo parâmetro do tipo ponteiro é por padrão não-nulo e input.

```
void F(T* p)
{
  printf("%d", p->i);
}
```

Qualquer comparação por nulo, torna o ponteiro opcional.


Qualquer atribuição torna o ponteiro out. Qualquer uso antes da atribuição torna o ponteiro in-out.


```
void Get(int *p)
{
   *p = 1; //out
}
```

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





