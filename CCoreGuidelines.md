

Result
Código de erros do aplicativo.

Semântica de Objetos

```
Result X_Init(X* px);
```

Inicializa o objeto X.  
Caso tenha sucesso na inicialização a finalização (X_Destroy) deve ser chamada. 
Em caso de erro, a instância não pode ser usada e nem destruída. 
Em caso de erro nenhum leak ou efeito colateral vai ser criado.
px nunca é nulo garantido pelo chamador.

void X_Destroy(X* px);

Destroi o objeto px. Se px for nulo não faz nada.
Depois da chamada desta função px não pode mais ser usado com exceção da função X_Init.
```
X*  X_Create();
```
Criar e inicializa um novo objeto X. A implementação é fixa e pode variar em relação ao alocador usado.
Implementação típica.
```
X* X_Create()
{
  X* px = (X*) malloc(sizeof(X) * 1);
  if (px != NULL)
  {
     Result result = X_Init(px);
     if (result != RESULT_OK)
     {
       free(px);
       px = NULL;
     }
  }
  return px;
}
```

```
void X_Delete(X* px)
```

Destroi o objeto px e devolve a memória para o alocador.
Se px for nulo a função não tem efeito.

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


Parâmetros de funções.

Todo parâmetro do tipo ponteiro é por padrão não-nulo e input.

Exemplo:

```
void F(X* px)
{
  //Nada declarado então px é IN não nulo.
}
```

Parâmetros do tipo ponteiro out deve ser comentados no momento da atribuição. 
Por padrão ele é considerado não nulo.
```
void Get(int *p)
{
   *p = 1; //out
}
```

Parâmetros IN opcionais devem ser comentados no momento do if.

```
void F(X* px)
{
  if (px != NULL) //optional
  {
    ... usar px
  }
}
```

Todos os parâmetros ponteiros são considerados não donos do conteúdo a não ser que informe o contrário.
Caso a função receba um ponteiro da qual é dona ela deve informar no momento da destruição ou no momento da trasferência de ownership.



```
void F(X** ppx)
{
  X* px = *ppx;
  *ppx = NULL;//moved

  X_Delete(px); //owner
}
```


Para casos exceptionais de performance pode-se ignorar a atruibição para nulo.

```
void F(X* px)
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










