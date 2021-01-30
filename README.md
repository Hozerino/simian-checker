# Simian-Checker

Simian-Checker é uma aplicação escrita em Java + SpringBoot e é capaz de verificar se uma determinada sequência de DNA pertence a um ser Humano ou Mutante.  
Resumidamente, a aplicação utiliza o MongoDB para armazenar os DNAs já computados e evitar que sejam computados novamente. A app também possui testes unitários e de integração, contando com mais de 90% de code coverage nos testes.

## Banco
Para persistir informações de DNAs já computados, o Simian-Checker utiliza um MongoDB; banco não-relacional escolhido devido à sua rápida execução de SELECTs e INSERTs. Isso é importante pois a aplicação não conta com DELETEs nem UPDATEs, uma vez que os DNAs são computados e gravados no banco ou apenas buscados do banco caso já existam.

## Frameworks e dependências
A aplicação, como dito anteriormente foi escrita em Java 8 + SpringBoot, a ferramenta de build utilizada foi o Gradle.

A aplicação também utiliza:  
* JUnit - testes unitários
* Mockito - mockar componentes para testes
* Swagger - documentação (será abordado mais à frente)
* Lombok - ajuda a evitar código boilerplate como construtores, getters e setters etc
* EmbeddedMongo - MongoDB embedded utilizado nos testes integrados


## Estrutura da aplicação
A aplicação foi divida principalmente em dois pacotes, o `rest` e o `domain` (também tem o common, mas não é tão "principal" assim).
* rest - contém tudo que diz respeito à camada REST, como anotações de validação, classes de payload, controllers, converters, filters etc;
* domain - contém o `core` da aplicação, como entidades de banco, services, repositórios, exceptions etc;
* common - contém tudo que é utilizado em ambos os pacotes, nesse caso, só contém a lógica de validação de DNA (se a matriz é quadrada, caracteres etc), pois não é uma lógica única de uma camada ou de outra.  
  
A ideia de separar as camadas de tal maneira é para que a camada de aplicação seja independente da camada de interface. Com isso podemos utilizar a lógica da aplicação e mudar o ponto de entrada, como deixar de usar REST e passar a utilizar algum CLI, ou até mesmo outra tecnologia como gRPC, pois só deveria ser reimplementada a lógica da interface.  
Por outro lado, a camada REST conhece alguns objetos da camada de service, pois a camada da interface é quem faz a comunicação do mundo de fora (cliente) com o mundo de dentro (aplicação), mas a aplicação não deve conhecer entidades do mundo "de fora".

![Alt text here](images/simian.png)

## Como a aplicação funciona?
### Cálculo do `isSimian(String[] dna)`
O ponto de entrada da aplicação para determinar se um DNA é simio ou não é na classe `SimianCheckerController`; antes que o objeto de request entre na lógica da aplicação, há uma anotação `@ValidDNAMatrix` que validará se o DNA enviado é válido (matrix quadrada, caracteres permitidos etc); caso não seja, a aplicação retornará status 400 e um payload indicando os erros do Request.  
Após isso, o DNA é então enviado ao `OrchestratorService`, service responsável por verificar se o DNA já foi computado ou se deve passar pelo algoritmo de identificação (OBS.: Esse service é cacheado por 15 minutos para evitar hits desnecessários no banco).

O algoritmo `isSimian(String[] dna)` está no service `SimianCheckerService` que é responsável unicamente por identificar se um DNA é simio ou não. Mas antes de iniciar o algoritmo, ele faz uma chamada a um service que faz a validação do DNA (mesmo já tendo sido validado na camada REST), pois como o Service é agnóstico do que a camada de interface faz ou deixa de fazer, ele deve validar o que entra para evitar um erro 500. Então finalmente, ele inicia o algoritmo real.

Sua lógica consiste em quebrar o problema em 3 etapas:
* Verificar se há algum padrão horizontal;
  * Para cada linha da matriz, ele busca iterativamente uma repetição de 4 caracteres.
* Verificar se há algum padrão vertical;
  * Para cada coluna da matriz, ele busca iterativamente uma repetição de 4 caracteres.
* Verificar se há algum padrão diagonal;
  * Para cada caractere da matriz, ele chama 2 métodos recursivos, um para a esquerda e um para a direita buscando algum match de caractere igual e incrementando um contador passado como parâmetro da chamada.

Apesar da recursão ser o pesadelo de muita gente, como o máximo de chamadas empilhadas é 2*4 (esquerda e direita * 4 caracteres no máximo), não gerará problemas como StackOverflow ou afins.  
A complexidade deste algoritmo, por percorrer a matriz praticamente 4 vezes (transformar em uma matriz de char, verificações vertical, horizontal e diagonal percorrendo cada char), é cerca de ```O(4n² + m)``` utilizando n=ordem da matriz e m=caracteres a mais percorridos na verificação diagonal, como m está limitado à ordem da matriz (m é da mesma ordem que n, ou seja, é impossível que ```m=n²```), podemos desprezá-lo e, simplificando a demonstração final, temos ```O(n²)```.

Por fim, após o algoritmo verificar se o DNA é simio ou não, o DNA é digerido em um SHA-1 para que seja salvo no banco sem que ocupe um espaço fortemente variável e que facilite a comparação; e então o Controller faz a verificação se deve retornar 200 ou 403, afinal, a camada REST e ela apenas, tem a responsabilidade sobre como a resposta será montada.

### Endpoint de /stats
O endpoint de statistics é bem simples, o StatsController faz a requisição para o StatsService que, por sua vez, utiliza o DnaRepository para que busque no banco a quantidade de DNAs salvos com cada tipo (humano ou mutante), retorna um objeto para o Controller e o Controller , por ser da camada de interface REST, faz a tratativa no objeto e o converte para o objeto de response, podendo omitir campos se precisar.



## Como rodar a aplicação
Para rodar a aplicação, há, a grosso modo, 2 maneiras:
1) Aplicação em Docker apontando para o banco "produtivo" (será explicado posteriormente);
2) Aplicação e banco ambos Dockerizados.

### Aplicação dockerizada apontando para o banco remoto
Para executar a aplicação em docker (expondo a porta 80) e fazê-la apontar para o banco remoto, basta executar o seguinte comando:
```bash
docker run -p 80:8080 -e "SPRING_PROFILES_ACTIVE=prod" dockerpoc:latest 
```

### Aplicação e banco dockerizados
Para executar tudo em docker, você pode utilizar o docker-compose com os seguintes comandos:
```bash
./gradlew clean build                  # gera o entregável da aplicação
docker build -t simian-checker-app .   # monta a imagem Docker que será utilizada pelo compose
docker-compose up                      # sobe o banco e a aplicação, ambos dockerizados
```

## Como consumir da aplicação
Apesar de ser possível rodar a aplicação local (como indicado anteriormente), também é possível consumí-la da internet, onde a aplicação está exposta.  
* API - a API pode ser acessada via
```
https://simian-checker-app.herokuapp.com/
```

* MongoDB - o banco  está hospedado no Atlas. Para consultá-lo, pode usar a seguinte String de conexão no Compass ("Workbench" que utilizei para acessar o MongoDB). O nome do db é `simian_checker_db` e a collection é `dna`
```
mongodb+srv://meli_user:meli_passwd@simian-checker-cluster.auxqr.mongodb.net/simian_checker_db
```

### Documentação da API
A documentação, como indicado no início, está disponível no Swagger da aplicação:
```
https://simian-checker-app.herokuapp.com/swagger-ui.html
```
