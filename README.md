### Como resolver uma colisão de artefatos de versão no Maven

# 1. Visão Geral
Projetos Maven com vários módulos podem ter gráficos de dependência complexos. Eles podem ter resultados incomuns, quanto mais os módulos são importados uns dos outros.

Neste tutorial, veremos como resolver a colisão de versões de artefatos no Maven.

Começaremos com um projeto de vários módulos em que usamos deliberadamente diferentes versões do mesmo artefato. Em seguida, veremos como evitar a obtenção da versão errada de um artefato com o gerenciamento de exclusão ou dependência.

Finalmente, tentaremos usar o plug-in maven-enforcer para tornar as coisas mais fáceis de controlar, banindo o uso de dependências transitivas.

# 2. Colisão de versões de artefatos
Cada dependência que incluímos em nosso projeto pode se vincular a outros artefatos. O Maven pode trazer automaticamente esses artefatos, também chamados de dependências transitivas. A colisão de versões ocorre quando várias dependências se vinculam ao mesmo artefato, mas usam versões diferentes.

Como resultado, pode haver erros em nossos aplicativos, tanto na fase de compilação quanto no tempo de execução.

### 2.1. Estrutura do Projeto
Vamos definir uma estrutura de projeto multi-módulo para experimentar. Nosso projeto consiste em um pai de colisão de versão e três módulos filhos:

```
version-collision
    project-a
    project-b
    project-collision
```

O pom.xml para project-a e project-b são quase idênticos. A única diferença é a versão do artefato com.google.guava do qual eles dependem. Em particular, o project-a usa a versão 22.0:

```
<dependencies>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>22.0</version>
    </dependency>
</dependencies>
```

Mas, o projeto-b usa a versão mais recente, 29.0-jre:

```
<dependencies>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>29.0-jre</version>
    </dependency>
</dependencies>
```

O terceiro módulo, projeto-colisão, depende dos outros dois:

```
<dependencies>
    <dependency>
        <groupId>com.isaccanedo</groupId>
        <artifactId>project-a</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.isaccanedo</groupId>
        <artifactId>project-b</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
</dependencies>
```

Então, qual versão do guava  estará disponível para colisão de projetos?

### 2.2. Usando recursos da versão de dependência específica
Podemos descobrir qual dependência é usada criando um teste simples no módulo de colisão do projeto que usa o método Futures.immediateVoidFuture do guava:

```
@Test
public void whenVersionCollisionDoesNotExist_thenShouldCompile() {
    assertThat(Futures.immediateVoidFuture(), notNullValue());
}
```

Este método está disponível apenas na versão 29.0-jre. Nós herdamos isso de um dos outros módulos, mas só podemos compilar nosso código se obtivermos a dependência transitiva do projeto-b.

### 2.3. Erro de compilação causado por colisão de versões
Dependendo da ordem das dependências no módulo de colisão do projeto, em certas combinações o Maven retorna um erro de compilação:

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.1:testCompile (default-testCompile) on project project-collision: Compilation failure
[ERROR] /tutorials/maven-all/version-collision/project-collision/src/test/java/com/isaccanedo/version/collision/VersionCollisionUnitTest.java:[12,27] cannot find symbol
[ERROR]   symbol:   method immediateVoidFuture()
[ERROR]   location: class com.google.common.util.concurrent.Futures
```

Esse é o resultado da colisão de versões do artefato com.google.guava. Por padrão, para dependências no mesmo nível em uma árvore de dependências, o Maven escolhe a primeira biblioteca que encontra. Em nosso caso, ambas as dependências com.google.guava estão na mesma altura e a versão mais antiga é escolhida.

### 2.4. Usando o plugin maven-dependency
O plug-in maven-dependency é uma ferramenta muito útil para apresentar todas as dependências e suas versões:

```
% mvn dependency:tree -Dverbose

[INFO] --- maven-dependency-plugin:2.8:tree (default-cli) @ project-collision ---
[INFO] com.isaccanedo:project-collision:jar:0.0.1-SNAPSHOT
[INFO] +- com.isaccanedo:project-a:jar:0.0.1-SNAPSHOT:compile
[INFO] |  \- com.google.guava:guava:jar:22.0:compile
[INFO] \- com.isaccanedo:project-b:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- (com.google.guava:guava:jar:29.0-jre:compile - omitted for conflict with 22.0)
```

O sinalizador -Dverbose exibe artefatos conflitantes. Na verdade, temos uma dependência com.google.guava em duas versões: 22.0 e 29.0-jre. Este último é o que gostaríamos de usar no módulo de colisão do projeto.

# 3. Excluindo uma dependência transitiva de um artefato
Uma maneira de resolver uma colisão de versão é removendo uma dependência transitiva conflitante de artefatos específicos. Em nosso exemplo, não queremos que a biblioteca com.google.guava seja temporariamente adicionada do projeto - um artefato.

Portanto, podemos excluí-lo no pom de colisão do projeto:

```
<dependencies>
    <dependency>
        <groupId>com.isaccanedo</groupId>
        <artifactId>project-a</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <exclusions>
            <exclusion>
                <groupId>com.google.guava</groupId>
                <artifactId>guava</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>com.isaccanedo</groupId>
        <artifactId>project-b</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
</dependencies>
```

Agora, quando executamos o comando dependency: tree, podemos ver que ele não está mais lá:

```
% mvn dependency:tree -Dverbose

[INFO] --- maven-dependency-plugin:2.8:tree (default-cli) @ project-collision ---
[INFO] com.isaccanedo:project-collision:jar:0.0.1-SNAPSHOT
[INFO] \- com.isaccanedo:project-b:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- com.google.guava:guava:jar:29.0-jre:compile
```

Como resultado, a fase de compilação termina sem erros e podemos usar as classes e métodos da versão 29.0-jre.

# 4. Usando a seção dependencyManagement
A seção dependencyManagement do Maven é um mecanismo para centralizar as informações de dependência. Um de seus recursos mais úteis é controlar versões de artefatos usados como dependências transitivas.

Com isso em mente, vamos criar uma configuração de dependencyManagement em nosso pom pai:

```
<dependencyManagement>
   <dependencies>
      <dependency>
         <groupId>com.google.guava</groupId>
         <artifactId>guava</artifactId>
         <version>29.0-jre</version>
      </dependency>
   </dependencies>
</dependencyManagement>
```

Como resultado, o Maven usará a versão 29.0-jre do artefato com.google.guava em todos os módulos filhos:

```
% mvn dependency:tree -Dverbose

[INFO] --- maven-dependency-plugin:2.8:tree (default-cli) @ project-collision ---
[INFO] com.isaccanedo:project-collision:jar:0.0.1-SNAPSHOT
[INFO] +- com.isaccanedo:project-a:jar:0.0.1-SNAPSHOT:compile
[INFO] |  \- com.google.guava:guava:jar:29.0-jre:compile (version managed from 22.0)
[INFO] \- com.isaccanedo:project-b:jar:0.0.1-SNAPSHOT:compile
[INFO]    \- (com.google.guava:guava:jar:29.0-jre:compile - version managed from 22.0; omitted for duplicate)
```

# 5. Previna dependências transitivas acidentais
O plug-in maven-enforcer fornece muitas regras integradas que simplificam o gerenciamento de um projeto com vários módulos. Um deles proíbe o uso de classes e métodos de dependências transitivas.

A declaração de dependência explícita remove a possibilidade de colisão de versões de artefatos. Vamos adicionar o plug-in maven-enforcer com essa regra ao nosso pom pai:

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.0.0-M3</version>
    <executions>
        <execution>
            <id>enforce-banned-dependencies</id>
            <goals>
                <goal>enforce</goal>
            </goals>
            <configuration>
                <rules>
                    <banTransitiveDependencies/>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Como consequência, agora devemos declarar explicitamente o artefato com.google.guava em nosso módulo de colisão de projeto se quisermos usá-lo nós mesmos. Devemos especificar a versão a ser usada ou configurar o dependencyManagement no pom.xml pai. Isso torna nosso projeto mais à prova de erros, mas exige que sejamos mais explícitos em nossos arquivos pom.xml.

# 6. Conclusão

Neste artigo, vimos como resolver uma colisão de versões de artefatos no Maven.

Primeiro, exploramos um exemplo de uma colisão de versões em um projeto de vários módulos.

Em seguida, mostramos como excluir dependências transitivas no pom.xml. Vimos como controlar as versões das dependências com a seção dependencyManagement no pom.xml pai.

Finalmente, tentamos o plug-in maven-enforcer para banir o uso de dependências transitivas para forçar cada módulo a assumir o controle por conta própria.