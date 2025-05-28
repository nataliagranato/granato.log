# granato.log:  O Problema das Configurações Hardcoded
### Use variáveis de ambiente para tornar seu código mais seguro e portável

*Por: Natália Granato*  
*Publicado em: 27 de maio de 2025*

## O Problema

Tudo começou durante um dia de trabalho com um projeto experimental de API de métricas para Kubernetes que estou desenvolvendo. A ideia surgiu porque eu não queria depender de uma plataforma completa de observabilidade para obter informações básicas. O projeto não possui dependências externas, sendo totalmente autossuficiente e leve. Por exemplo, sequer há necessidade de instalação do `metrics-server`; não é preciso Prometheus, Grafana ou qualquer outra ferramenta. É uma API simples que expõe métricas básicas do cluster Kubernetes, como uso de CPU, memória e status dos pods, com autenticação via token.

E, claro, existe um endpoint `/metrics` que retorna as métricas em formato JSON, e outro, `/metrics-prometheus`, que as retorna no formato Prometheus, caso eu queira integrá-lo com outras ferramentas de monitoramento.

Durante os testes, percebi um comportamento estranho: mesmo utilizando o token de autenticação correto, a API continuava rejeitando todas as requisições com a mensagem de erro: "token inválido".

No terminal, o comando parecia perfeitamente correto:

```bash
curl -H "Authorization: Bearer meuTokenSuperSeguro123!@#" http://localhost:8080/metrics
```

A resposta era sempre a mesma:

```
Acesso não autorizado: token inválido
```

Comecei a questionar se os valores estavam sendo passados corretamente para o contêiner, visto que havia realizado o *deployment* no KinD com o chart desenvolvido para a aplicação. O token estava sendo enviado corretamente no header de autorização, o formato "Bearer" estava presente, e eu estava usando exatamente o mesmo token que havia configurado no Secret do Kubernetes para o projeto experimental.

## A Investigação

Decidi investigar um pouco mais. Uma boa maneira de começar, nesse caso, é verificar os Secrets do Kubernetes. Por padrão, os dados dentro de um Secret são armazenados codificados em Base64 (o Kubernetes os decodifica ao injetá-los como variáveis de ambiente ou arquivos nos pods). A primeira coisa que fiz foi verificar se o token estava realmente chegando à aplicação. Entrei no pod da API para examinar a variável de ambiente:

```bash
kubectl exec -it -n k8s-api-metrics \
  $(kubectl get pods -n k8s-api-metrics -o jsonpath='{.items[0].metadata.name}') -- sh

# E dentro do pod utilizei o comando:
echo "$EXPECTED_AUTH_TOKEN"
```

Para minha surpresa, o token estava sendo recebido corretamente no pod, mas havia um problema: ele tinha uma quebra de linha no final! Não era visível diretamente, mas estava lá, um caractere `\n` indesejado.

No entanto, o problema era ainda mais profundo. Quando analisei o código da minha aplicação, descobri que o token nem estava sendo lido da variável de ambiente! O token esperado estava *hardcoded* diretamente no código, em vez de usar o valor que vinha do Secret do Kubernetes.

## O Código Problemático

Olhando para o código-fonte da API, encontrei dois problemas críticos:

```go
// Variável global definida com valor fixo
var expectedAuthToken = "tokenFixoHardcoded123" // PROBLEMA 1: Token hardcoded

// Middleware de autenticação
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Acesso não autorizado: token não fornecido", http.StatusUnauthorized)
            return
        }

        // Espera "Bearer <token>"
        parts := strings.Split(token, " ")
        if len(parts) != 2 || strings.ToLower(parts[0]) != "bearer" {
            http.Error(w, "Acesso não autorizado: formato de token inválido", http.StatusUnauthorized)
            return
        }
        actualToken := parts[1]

        // PROBLEMA 2: Comparação direta sem tratar quebras de linha
        if actualToken != expectedAuthToken {
            http.Error(w, "Acesso não autorizado: token inválido", http.StatusUnauthorized)
            return
        }

        next.ServeHTTP(w, r)
    }
}
```

Identifiquei dois problemas graves:

1. O token esperado estava definido como uma variável global com um valor fixo (*hardcoded*), ignorando completamente a variável de ambiente `EXPECTED_AUTH_TOKEN` que estava configurada no Kubernetes.
2. Mesmo se estivesse lendo a variável de ambiente, a comparação direta `actualToken != expectedAuthToken` não tratava possíveis quebras de linha ou espaços.

## A Solução

A solução envolveu duas mudanças fundamentais no código:

No `main.go` (ou onde a configuração da aplicação é inicializada):

```go
import (
	"log"
	"net/http"
	"os"
	"strings"
)

// Alterando a variável global para não ter valor fixo
var expectedAuthToken string // Agora é uma string vazia, sem valor hardcoded

// Exemplo de handler, substitua pelo seu
func metricsHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("Métricas aqui!"))
}

func main() {
	// Configuração inicial...

	// 1. Lê o token de autenticação da variável de ambiente
	tokenFromEnv := os.Getenv("EXPECTED_AUTH_TOKEN")
	if tokenFromEnv == "" {
		// Idealmente, use um logger mais robusto aqui
		log.Fatal("ERRO: Variável de ambiente EXPECTED_AUTH_TOKEN não definida.")
		// os.Exit(1) é implícito com log.Fatal()
	}

	// 2. Remove qualquer quebra de linha ou espaço em branco do token e atribui à variável global
	expectedAuthToken = strings.TrimSpace(tokenFromEnv)

	// Restante da inicialização da aplicação...
	// Exemplo:
	http.HandleFunc("/metrics", authMiddleware(metricsHandler))
	log.Println("Servidor escutando na porta 8080...")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Essa solução resolveu os dois problemas:

- Agora o token esperado é lido da variável de ambiente `EXPECTED_AUTH_TOKEN` no início da aplicação, em vez de usar um valor fixo no código.
- A função `strings.TrimSpace()` remove quaisquer espaços em branco ou quebras de linha indesejados do token lido do ambiente, garantindo uma comparação limpa.

## Lições Aprendidas

Este problema reforçou algumas lições importantes sobre práticas de desenvolvimento e infraestrutura:

- **Nunca use valores hardcoded** para autenticação ou qualquer configuração sensível/variável. Sempre leia credenciais e configurações de variáveis de ambiente, arquivos de configuração externos ou sistemas de gerenciamento de segredos.
- **Atenção aos caracteres invisíveis.** Caracteres como quebras de linha (`\n`) ou espaços extras no início/fim de strings podem causar problemas difíceis de diagnosticar, especialmente com dados provenientes de fontes externas ou processos de codificação/decodificação.
- **Valide e normalize entradas.** Nunca confie em comparações diretas de strings para autenticação ou lógica crítica sem antes garantir que ambas as strings estão no formato esperado (por exemplo, trimando espaços, convertendo para minúsculas/maiúsculas se for o caso).
- **A criação de Secrets no Kubernetes exige cuidado.** A forma como os valores são gerados e codificados antes de serem aplicados no Kubernetes pode introduzir caracteres extras, como quebras de linha (por exemplo, usar `echo -n "valor"` em vez de `echo "valor"` ao canalizar para base64).
- **Logs e ferramentas de depuração são essenciais.** Sem a capacidade de inspecionar o ambiente de execução (como entrar no pod e verificar o valor exato da variável de ambiente), a resolução de problemas como este pode consumir muito mais tempo.

## Mudanças de Infraestrutura

Além da correção no código, também melhorei a documentação e o processo de deploy do meu projeto experimental:

- Atualizei o `README.md` para incluir informações sobre a configuração do token e possíveis problemas com caracteres extras.
- Construí uma nova versão da imagem Docker (v1.0.1) com a correção.
- Atualizei o Helm chart para usar a nova versão da imagem e para documentar melhor como o Secret do token deve ser criado.

## Conclusão

O conhecimento que reforcei com este processo experimental é que nunca é uma boa ideia fazer *hardcode* de valores sensíveis ou dinâmicos diretamente no código. Isso não só torna o código menos flexível, mas também pode introduzir vulnerabilidades de segurança e bugs difíceis de rastrear.

Não é apenas uma questão de boas práticas, mas uma necessidade para aplicações *Cloud Native* modernas. Precisamos ter a flexibilidade de adaptar nossas aplicações a ambientes dinâmicos, como o Kubernetes, onde as configurações podem mudar rapidamente entre diferentes *deployments* e ambientes.

Esta é uma lembrança importante de que devemos seguir o princípio da "configuração externa" – nunca fazer *hardcode* de valores que podem mudar entre ambientes ou *deployments*. No Kubernetes, isso é especialmente importante, pois todo o ecossistema é construído em torno da ideia de injetar configurações através de ConfigMaps e Secrets ou utilizar soluções como external-secrets e Vault para um gerenciamento de segredos mais robusto.

A solução foi relativamente simples, mas o caminho para encontrá-la exigiu paciência, observabilidade a olho nu e um bom entendimento de como os dados fluem através dos diversos componentes da infraestrutura Kubernetes, desde o Helm chart até o contêiner em execução.

Como diz o manifesto do Twelve-Factor App: "**Armazene a configuração no ambiente**" (que implica configurar em tempo de execução, não em tempo de compilação). E essa lógica é fundamental para quem trabalha com DevOps e *Cloud Native*.

---

### Sobre mim

Sou Engenheira de Plataforma (*Platform Engineer*), especializada na construção de contêineres e orquestração com Kubernetes. Sou apaixonada por melhoria contínua, DevSecOps, boa documentação e, claro, observabilidade.

Este artigo faz parte da série "granato.log", onde compartilho histórias reais e lições aprendidas em minha jornada na tecnologia.

---

**Principais alterações e sugestões:**

- **Fluidez e Clareza:** Ajustei algumas frases para melhorar a fluidez e a clareza, dividindo sentenças longas e tornando a linguagem mais direta.
- **Termos Técnicos:** Usei itálico para termos estrangeiros consolidados como *hardcoded*, *Cloud Native* e *deployments* quando usados no corpo do texto. Mantive a capitalização original de ferramentas e tecnologias (`KinD`, `Kubernetes`, `Helm`, `Prometheus`, `Go`, etc.).
- **Precisão Técnica:**
    - Esclareci um pouco o funcionamento dos Secrets em Base64 no Kubernetes.
    - Na seção "Lições Aprendidas", refinei o ponto sobre a origem dos caracteres extras nos Secrets.
    - No bloco de código da solução, sugeri o uso de `log.Fatal` para consistência e adicionei um comentário sobre o `os.Exit(1)` ser implícito. Também mudei `expectedAuthToken = os.Getenv(...)` para `tokenFromEnv := os.Getenv(...)` e depois `expectedAuthToken = strings.TrimSpace(tokenFromEnv)` para garantir que a variável global `expectedAuthToken` sempre contenha o valor "trimado".
- **Concordância e Ortografia:** Corrigi pequenos deslizes de concordância e ortografia (ex: "melhoría" para "melhoria", "construção containeres" para "construção de contêineres").
- **Seção "Sobre mim":** Ajustei para "Engenheira de Plataforma" para concordância e refinei a lista de paixões para melhor paralelismo. Mudei "irei compartilhar" para "compartilho" para dar um tom mais presente à série.

Espero que esta revisão ajude a aprimorar ainda mais seu excelente artigo! É um ótimo exemplo prático de um problema comum e bem explicado.
