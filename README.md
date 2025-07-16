![](/static/site_logo.png)

# Blog do Projeto LLM-PT-IBM

Este repositório contém o código-fonte do blog do projeto *LLM-PT-IBM*, desenvolvido em parceria entre a **Universidade Federal de Campina Grande (UFCG)** e a **IBM**. O blog apresenta relatórios e resultados relacionados à avaliação de modelos de linguagem para a língua portuguesa. Aqui, você encontrará todo o conteúdo do blog, incluindo posts, análises e atualizações sobre os avanços do projeto. Para visualizar o blog em funcionamento, acesse este [link](https://llm-pt-ibm.github.io/).

Este blog foi desenvolvido utilizando o Hugo, um framework de geração de sites estáticos rápido e flexível, baseado em Go. Para mais informações, visite a documentação oficial: https://gohugo.io/.

A seguir, você encontrará um tutorial com instruções para executar o blog localmente e adicionar uma nova publicação.

## Executando o blog localmente

Você pode executar o blog de duas formas: instalando o Hugo diretamente ou utilizando Docker.

### Executando o Servidor Hugo

Certifique-se de que o Hugo está instalado em sua máquina. Para verificar, execute:

```sh
hugo version
```

Se o Hugo não estiver instalado, consulte a [documentação oficial](https://gohugo.io/getting-started/installing/) para instalá-lo.

#### Passos para executar o servidor Hugo


1. Navegue até a pasta raiz do projeto e execute o servidor localmente:
   ```sh
   hugo server
   ```

2. O servidor será iniciado e estará disponível em uma das portas locais. A URL será exibida no terminal, como no exemplo abaixo:
   ```
   http://localhost:1313
   ```
### Executando o Servidor Hugo com Docker

Caso prefira usar o Docker, siga os passos abaixo. Primeiramente, verifique se o Docker está instalado em sua máquina:

```sh
docker --version
```

Se o Docker não estiver instalado, siga as instruções da [documentação oficial do Docker](https://docs.docker.com/get-docker/) para instalá-lo.

#### Passos para executar o Hugo com Docker

1. No terminal, execute o comando abaixo para construir a imagem Docker a partir do Dockerfile:
   ```bash
   sudo docker build -t hugo-server .

2. Após a imagem ser construída, execute o seguinte comando para iniciar o contêiner com o Hugo:
   ```
   sudo docker run --rm -p 1313:1313 -v $(pwd):/site hugo-server
   ```

3. Após rodar o container, você poderá acessar seu site Hugo no navegador em:
   ```
   http://localhost:1313
   ```

## Adicionando uma publicação ao blog

1. Crie uma nova _branch_ no repositório.
2. Adicione o conteúdo em português que deseja publicar em um arquivo no formato Markdown (`.md`) dentro do diretório `/content/pt/posts/`.
3. Adicione o conteúdo em inglês que deseja publicar em um arquivo no formato Markdown (`.md`) dentro do diretório `/content/en/posts/`.
   1. É importante que tenha seu conteúdo seja feito nas duas versões e o arquivo (`.md`) tenham o mesmo nome.
3. Execute o blog localmente para garantir que as alterações foram aplicadas corretamente.
4. Crie um _pull request_ para mesclar a sua _branch_ com a branch principal do repositório.
5. Aguarde o _pull request_ ser revisado e aceito. Após a aprovação, as alterações serão publicadas no blog.

Um exemplo de _template_ para publicação está disponível abaixo.
```
---
title: "Título da publicação"
date: 2025-03-12 # ano-mês-dia
authors: ["Nome do Autor", "Outro Autor"] # Pode ser uma lista
tags: ["LLM", "benchmark", "pesquisa"]
translationKey: "nome-do-post"
summary: "Breve resumo do post, que aparecerá como descrição no blog."
draft: false # Mude para true se quiser que o post fique como rascunho
---

## Contexto  
Breve explicação sobre o que está sendo abordado no post. Pode incluir o problema ou desafio inicial e sua relevância para a equipe.

## Metodologia / Processo  
Descrição do que foi feito, quais técnicas, ferramentas ou benchmarks foram utilizados. Pode incluir detalhes sobre modelos, datasets e métricas.

## Resultados e Conclusões  
Resumo dos achados, principais insights e impactos dos experimentos ou do desenvolvimento realizado. Pode incluir tabelas, gráficos ou comparações.

## Próximos Passos  
O que será feito a seguir? Há melhorias planejadas, novos experimentos ou desafios a serem enfrentados?

## Recursos e Referências  
Links úteis, papers, repositórios ou outras referências que sejam relevantes para quem quiser se aprofundar no tema.
```

### Adicionando imagens nas publicações

No Hugo, a maneira mais simples de incluir imagens nas suas publicações é armazená-las na pasta ``/static``. Qualquer arquivo colocado nesta pasta será acessível diretamente através da URL do seu site.

Para adicionar uma imagem no seu post, basta utilizar a seguinte sintaxe em Markdown, substituindo ``minha-imagem.jpg`` pelo nome do arquivo que você colocou na pasta ``/static``:

```markdown
{{< figure src="/images/minha-imagem.png" alt="Descrição alternativa" caption="Legenda da Imagem" >}}
```

### Adicionando links nas publicações
Para adicionar um link externo e seguir a padronização do blog com a cor do link, basta seguir esse formato: 

```markdown
[<span class="link-personalizado">Texto para o link</span>](https://exemplo.com.br)
```
A class "link-personalizado" no arquivo ``styles-light.css`` irá personalizar o link com a cor padrão. 

Para adicionar um link que remete a alguma outra postagem do blog, basta seguir esse formato:
[<span class="link-personalizado">Texto para o link</span>]({{< relref "nome_de_outro_post.md" >}})
