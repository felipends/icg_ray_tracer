# Atividade 5 - Introdução ao _Ray tracing_

### José Felipe Nunes da Silva - 201700196010
### Rebeca Raab Bias Ramos - 20170070453
---

## Introdução

É proposto neste trabalho a implementação do termo especular do modelo de iluminação de _Phong_ para uma rotina de _Ray Trancing_, tendo como ponto inicial a geometria de uma esfera para conferir resultados. Sendo o segundo objetivo a implementação do método que descreve o comportamento da intersecção de raios de luz com um triângulo.

Os experimentos foram desenvolvidos utilizando a linguagem de programação JavaScript, a partir do _framework_ fornecido pelo professor da disciplina, com o suporte da biblioteca *THREE.js* para realizar operações envolvendo vetores.

Desenvolvimento e resultados são discutidos nas seções a seguir.

## Termo especular no modelo de iluminação de Phong

Também chamado de _Phong Shading_, este modelo de iluminação é descrito algebricamente por uma expressão com três termos:

* Termo ambiente: $I_{a}K_{a}$
* Termo difuso: $I_{p}K_{d}(L \cdot N)$
* Termo especular: $I_{p}K_{s}(R \cdot V)^n$

Somados: 

$I = I_{a}K_{a} + I_{p}K_{d}(L \cdot N) + I_{p}K_{s}(R \cdot V)^n$

Onde:

* $I$: intensidade (cor) final calculada para o fragmento.
* $I_{a}$: intensidade da luz ambiente.
* $K_{a}$: coeficiente de reflectância ambiente.
* $I_{p}$: intensidade da luz pontual/direcional.
* $K_{d}$: coeficiente de reflectância difusa.
* $N$: vetor normal.
* $L$: vetor que aponta para a fonte de luz pontual/direcional.
* $K_{s}$: coeficiente de reflectância especular.
* $R$: reflexão de $L$ sobre $N$.
* $V$: vetor que aponta para a câmera.
* $n$: tamanho do brilho especular.

No _framework_ fornecido como base para o desenvolvimento deste trabalho são inclusos os demais termos da expressão com exceção do termo especular, responsável pelo cálculo do efeito conhecido como _highlight_, da iluminação sobre a geometria. A implementação deste termo resume-se ao cálculo dos vetores $R$, $V$, a verificação do ângulo entre eles, através do produto interno, este resultado deve ser elevado a um valor estático arbitrário $n$, que define o tamanho do _highlight_.  Por fim, multiplica-se o valor obtido por $Ip$ e $K_s$. 

Com os demais termos já calculados, o cálculo do termo especular é realizado da seguinte forma no trecho do código fonte abaixo: 

```js
// Calculo do termo especular

// Expoente que define tamanho do brilho especular.
let n = 32; 
// Vetor de reflexão da luz.
let R = L.clone().reflect(interseccao.normal).negate().normalize();
// Vetor que aponta para a câmera (que encontra-se na origem).
let V = interseccao.posicao.clone().negate().normalize();  
let termo_especular = Ip.cor.clone().multiply(ks).multiplyScalar(Math.pow(Math.max(0.0, V.dot(R)), n));
```

É importante notar neste trecho que:

1. O vetor $R$ é a reflexão do vetor $L$ em relação à normal da intersecção do raio com a superfície. Contudo, ao realizar este cálculo utilizando o método `reflect()` da biblioteca `THREE.js` o resultado é um vetor na direção oposta da desejada, sendo necessário a utilização do método `negate()`, invertendo a direção do vetor.

2. O vetor $V$ é aquele que, partindo da geometria, aponta para a câmera. Levando em consideração que a câmera encontra-se na origem e que o ponto de onde parte o vetor é a posição da intersecção do raio com a geometria, basta que seja invertida a direção do vetor da posição da intersecção.

É importante ressaltar ainda os parâmetros que foram fornecidos pelo professor para a realização de testes de funcionamento da implementação:

* $K_s = (1,1,1)$
* $n = 32$

O resultado obtido ao fim do processo de renderização é exibido na figura a seguir.

<img src="https://imgur.com/aPZPsEm.png" alt="esfera especular"  style="width:300px;" />

É notável a presença de um ponto mais luminoso na parte superior esquerda da esfera e o restante com tons mais escuros. 

## Rendering de triângulos

Diferentemente do processo de rasterização, para incluir um novo objeto na cena é necessário que seja descrito matematicamente como se dão as intersecções entre os raios de luz e a geometria. No _framework_ fornecido pelo professor é incluída uma classe que descreve como se dá a intersecção dos raios com uma esfera e é solicitado que seja implementada um método de renderização de triângulos. Desta maneira, foi implementada a classe `Triangulo`, com os dados `vertice0`, `vertice1` e `vertice2`. Além disso, baseando-se no modelo de intersecção proposto em [Möller and Trumbore, 2005](https://cadxfem.org/inf/Fast%20MinimumStorage%20RayTriangle%20Intersection.pdf), foi implementada a função `interseccionar`, para validar a intersecção entre raios e o triângulo, caso o valor retornado seja `true` o _pixel_ na posição da intersecção recebe uma cor referente ao valor calculado para aquele ponto pelo modelo de iluminação utilizado. Tal implementação é apresentada no trecho do código fonte a seguir.

```js
class Triangulo {
    constructor(vertice0, vertice1, vertice2) {
        this.vertice0 = vertice0;
        this.vertice1 = vertice1;
        this.vertice2 = vertice2;
    }

    interseccionar(raio, interseccao) {
        // arestas a partir do vertice 0
        let aresta0 = this.vertice1.clone().sub(this.vertice0);
        let aresta1 = this.vertice2.clone().sub(this.vertice0);

        // cálculo do determinante
        let pvec = raio.direcao.clone().cross(aresta1);
        let det = aresta0.clone().dot(pvec);

        // testes de intersecção
        if (det > -0.001 && det < 0.001)
            return false;

        let inv_det = 1.0 / det;

        let tvec = raio.origem.clone().sub(this.vertice0);

        let u = tvec.clone().dot(pvec) * inv_det;
        if (u < 0.0 || u > 1.0)
            return false;

        let qvec = tvec.clone().cross(aresta0);

        let v = raio.direcao.clone().dot(qvec) * inv_det;
        if (v < 0.0 || u + v > 1.0)
            return false;

        // calculo da posição da intersecção
        interseccao.t = aresta1.clone().dot(qvec) * inv_det;
        interseccao.posicao = raio.origem.clone().add(raio.direcao.clone().multiplyScalar(interseccao.t));
        // calculo da normal da intersecção
        interseccao.normal = aresta0.clone().cross(aresta1).negate().normalize();

        return true;

    }
}
```

Ao adicionar um objeto desta classe na função `Render()` (fornecida pelo professor no _framework_), com os vértices (−1.0, −1.0, −3.5), (1.0, 1.0, −3.0) e
(0.75, −1.0, −2.5), mantendo o $K_s = (1, 1, 1)$ e $n = 32$, obtemos este resultado da figura a seguir.

<img src="https://imgur.com/2gthJcb.png" alt="triangulo especular"  style="width:300px;" />

Nota-se a presença do triângulo na cena levemente inclinado e com brilho mais evidente do aquele presente na esfera.

## Referências

* Vídeo sobre _Ray tracing_ do professor Christian Pagot;
* Capítulos 6 do livro do Peter Shirley;
* Artigo _Fast, Minimum Storage Ray/Triangle Intersection_ por [Möller and Trumbore, 2005](https://cadxfem.org/inf/Fast%20MinimumStorage%20RayTriangle%20Intersection.pdf).