# Levantamento de bibliotecas de criptografia pós-quântica: ML-KEM e ML-DSA

Levantamento técnico de bibliotecas de código aberto que implementam os dois primeiros padrões pós-quânticos finalizados pelo NIST: o ML-KEM (FIPS 203), mecanismo de encapsulamento de chaves, e o ML-DSA (FIPS 204), esquema de assinatura digital. O documento compara cobertura funcional, conformidade com as normas, situação de certificação, garantias de verificação e otimizações por arquitetura de processador.

Última revisão: 16 de outubro de 2026.

---

## Sumário

1. [Motivação e escopo](#motivação-e-escopo)
2. [Bibliotecas analisadas](#bibliotecas-analisadas)
3. [Como ler as tabelas](#como-ler-as-tabelas)
4. [Cobertura e conformidade](#cobertura-e-conformidade)
5. [Otimizações por arquitetura](#otimizações-por-arquitetura)
6. [Desempenho](#desempenho)
7. [Estrutura do repositório](#estrutura-do-repositório)
8. [Referências](#referências)

---

## Motivação e escopo

A escolha de uma biblioteca de criptografia pós-quântica envolve mais do que a presença dos algoritmos: importam também a aderência às versões finais das normas, a existência de certificação, as garantias formais sobre o código e o suporte às plataformas de hardware de interesse. Este levantamento reúne essas informações de forma comparável.

Um dos usos considerados é a autenticação do canal clássico em sistemas de distribuição quântica de chaves (QKD), em que o ML-DSA atua na assinatura e o ML-KEM no estabelecimento de chaves. As referências sobre esse tema estão na última seção, e cada ficha individual traz uma seção dedicada a ele.

Os dados refletem as versões e os commits analisados em outubro de 2026, identificados na seção "Identificação" de cada ficha.

---

## Bibliotecas analisadas

A tabela a seguir apresenta as bibliotecas selecionadas. O nome de cada uma leva à respectiva ficha, que detalha os pontos resumidos neste documento.

| Biblioteca | Descrição | Linguagem | Licença |
| --- | --- | --- | --- |
| [mlkem-native e mldsa-native](Bibliotecas/mlkem-native%3Bmldsa-native.md) | Bibliotecas irmãs dedicadas exclusivamente a ML-KEM e ML-DSA, mantidas pela Post-Quantum Cryptography Alliance (Linux Foundation) | C90 + assembly | Apache-2.0 OR ISC OR MIT |
| [OpenSSL](Bibliotecas/OpenSSL.md) | Biblioteca de uso geral (libcrypto e libssl, com TLS, DTLS e QUIC); ML-KEM e ML-DSA nativos desde a 3.5.0 | C + assembly | Apache-2.0 |
| [wolfSSL](Bibliotecas/wolfSSL.md) | Biblioteca TLS/DTLS voltada a sistemas embarcados e de tempo real; a camada criptográfica é o wolfCrypt | C + assembly | GPLv3 ou comercial |
| [BoringSSL](Bibliotecas/BoringSSL.md) | Fork do OpenSSL mantido pelo Google para uso próprio, sem compromisso de API estável | C++ | Apache-2.0 |
| [Botan](Bibliotecas/Botan.md) | Biblioteca de uso geral com TLS 1.3 e X.509, desenvolvida com apoio do BSI (Alemanha) | C++20 | BSD-2-Clause |
| [leancrypto](Bibliotecas/leancrypto.md) | Biblioteca enxuta para espaço de usuário, kernel Linux, EFI e bare-metal | C + assembly | BSD-3-Clause ou GPLv2 |
| [IPP-Crypto](Bibliotecas/IPP-Crypto.md) | Primitivas criptográficas da Intel com despacho por geração de CPU, sem camada de protocolo | C | Apache-2.0 |

Dois projetos foram identificados como agregadores, isto é, para ML-KEM e ML-DSA eles incorporam implementações de terceiros em vez de manter código próprio: liboqs e AWS-LC. Ambos utilizam o mlkem-native e o mldsa-native e, por isso, não possuem ficha própria neste levantamento.

---

## Como ler as tabelas

As tabelas comparativas usam a notação de marcadores comum em artigos de sistematização (SoK), que separa de forma explícita o suporte completo, o parcial e a ausência de suporte.

| Marcador | Significado |
| --- | --- |
| ● | Suportado integralmente |
| ◐ | Suportado parcialmente: apenas um subconjunto (parâmetros, operações ou primitivas) ou somente em caráter experimental. A célula indica qual parte é coberta |
| ○ | Não suportado, ou não declarado na documentação do projeto |
| — | Valor ainda não medido |

Quando a característica não é binária (situação de certificação, método de verificação, licença), a célula traz o valor descritivo. Versões entre parênteses indicam a partir de qual release o recurso está disponível; "master" indica recurso presente apenas no ramo de desenvolvimento, ainda não lançado em versão estável.

---

## Cobertura e conformidade

Esta tabela verifica quais conjuntos de parâmetros e variantes de cada algoritmo estão disponíveis, o grau de aderência às versões finais das normas e as garantias oferecidas sobre a implementação. Vale distinguir dois programas do NIST citados abaixo: o CAVP valida a implementação de algoritmos individuais por meio de vetores de teste, enquanto o CMVP certifica o módulo criptográfico como um todo, segundo a FIPS 140-3.

| Característica | mlkem-native / mldsa-native | OpenSSL | wolfSSL | BoringSSL | Botan | leancrypto | IPP-Crypto |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ML-KEM-512/768/1024 | ● | ● (3.5+) | ● | ◐ apenas 768 e 1024 | ● | ● | ◐ API experimental |
| ML-DSA-44/65/87 | ● | ● (3.5+) | ● | ● | ● | ● | ◐ API experimental |
| HashML-DSA (pré-hash) | ● | ○ | ● | ○ | ○ | ● | ○ |
| External μ | ● | ● | ● | ● | ○ | ● | ◐ apenas na verificação |
| Conformidade com FIPS 203/204 finais | ● ACVP + Wycheproof | ● ACVP + Wycheproof | ● ACVP | ● vetores NIST + Wycheproof | ● ACVP (ML-KEM), Wycheproof (ML-DSA) | ● primeira validação CAVP do mundo (v1.0.0) | ● |
| Certificação CMVP (FIPS 140-3) | ○ | Em revisão (3.5.x) | Em auditoria final (v7.0.0) | Em revisão (v20251031) | ○ | ○ | ○ |
| Verificação formal | ● CBMC (C) e HOL-Light (assembly) | ○ | ○ | ○ | ○ | ○ | ○ |
| Verificação de tempo constante | Prova formal no assembly + valgrind | Dinâmica (valgrind) | Não declarada | Dinâmica (valgrind) | Dinâmica (valgrind) | Dinâmica (valgrind) | PQC fora dos testes |
| Licença | Apache-2.0 OR ISC OR MIT | Apache-2.0 | GPLv3 ou comercial | Apache-2.0 | BSD-2-Clause | BSD-3-Clause ou GPLv2 | Apache-2.0 |

Notas sobre a tabela:

- Certificação CMVP. No OpenSSL, o certificado ativo #4985 refere-se à versão 3.1.2, sem PQC; a 3.5.x foi submetida e está em revisão. No wolfSSL, os certificados #4718 e #5041 não incluem PQC; a v7.0.0, com PQC, está em auditoria final. No BoringSSL, o certificado #5296 não inclui PQC; a versão v20251031 está em revisão e já possui a validação CAVP A7700, que cobre os algoritmos pós-quânticos. A leancrypto tem validação CAVP e é estruturada como módulo FIPS, mas não possui certificado de módulo. A IPP-Crypto se declara "FIPS-compliant", sem certificado. Nenhum certificado existente inclui o código do mlkem-native ou do mldsa-native.
- Verificação formal (mlkem-native e mldsa-native). O CBMC prova segurança de memória e de tipos no código C. O HOL-Light, com o s2n-bignum, prova correção funcional, segurança de memória e tempo constante diretamente no código objeto, para AArch64 e x86_64 (e também RV32IM no mldsa-native). Os backends RVV, POWER e MVE não são cobertos pelas provas.
- IPP-Crypto. As funções de ML-KEM e ML-DSA só ficam disponíveis quando as macros `IPPCP_PREVIEW_ML_KEM` e `IPPCP_PREVIEW_ML_DSA` são definidas antes de incluir `ippcp.h`.
- BoringSSL. O ML-KEM-512 não está presente na API pública nem no módulo FIPS.

---

## Otimizações por arquitetura

Esta tabela indica, para cada arquitetura e extensão do conjunto de instruções, se a biblioteca possui código otimizado (assembly ou intrínsecos) além da implementação genérica em C. O custo de ML-KEM e ML-DSA concentra-se na transformada da teoria dos números (NTT) e na função Keccak (SHA-3/SHAKE); por isso, várias bibliotecas otimizam apenas uma dessas partes, e nesses casos a célula indica o que foi otimizado. O mlkem-native e o mldsa-native aparecem em colunas separadas porque seus backends diferem.

| Arquitetura | Extensão | mlkem-native | mldsa-native | OpenSSL | wolfSSL | BoringSSL | Botan | leancrypto | IPP-Crypto |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| x86_64 | AVX2 | ● | ● | ◐ NTT do ML-DSA (4.1+) | ● | ◐ Keccak x4 | ◐ ML-KEM e SHAKE x4 (master) | ● | ● |
| x86_64 | AVX-512 | ○ | ○ | ◐ SHAKE x4 do ML-DSA (4.1+) | ● inclui VBMI e Keccak x8 | ○ | ◐ Keccak | ◐ Keccak | ◐ ML-KEM |
| AArch64 | NEON | ● | ● | ○ | ◐ ML-KEM | ◐ Keccak x4 | ○ | ● | ○ |
| AArch64 | SVE/SVE2 | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| AArch64 | SHA3 (instruções Keccak) | ● | ● | ● | ● | ○ | ○ | ● inclui 2-way | ○ |
| RISC-V | RVV (vetorial) | ◐ NTT, apenas VLEN=256 | ○ | ○ | ◐ Keccak | ○ | ○ | ● VLEN 128 e 256 | ○ |
| RISC-V | Escalar otimizado | ○ | ● RV32IM | ○ | ◐ Keccak | ○ | ○ | ● RV64IM | ○ |
| POWER | VSX | ● ppc64le | ○ | ◐ NTT em ppc64le (4.1+) | ◐ Keccak | ○ | ○ | ○ | ○ |
| Armv8.1-M | Helium/MVE | ◐ experimental | ◐ experimental | ○ | ○ | ○ | ○ | ○ | ○ |

Notas sobre a tabela:

- OpenSSL. As otimizações marcadas com (4.1+) pertencem à série 4.1, ainda em beta na data desta revisão.
- Botan. As otimizações AVX2 existem apenas no ramo master, que dará origem à versão 3.14.
- Armv8.1-M. Nas duas bibliotecas do PQCP o backend MVE é experimental e desligado por padrão. O wolfSSL não usa MVE, mas oferece ML-KEM em assembly Thumb-2 escalar; a leancrypto oferece assembly para ARMv7 e Cortex-M.

---

## Desempenho

Esta seção está reservada para as medições comparativas, que ainda não foram coletadas. As tabelas definem as operações a medir para cada algoritmo; como o mlkem-native implementa apenas ML-KEM e o mldsa-native apenas ML-DSA, os dois algoritmos são tratados separadamente. A plataforma, o compilador, o conjunto de parâmetros e a unidade de medida serão registrados junto aos resultados.

ML-KEM:

| Operação | mlkem-native | OpenSSL | wolfSSL | BoringSSL | Botan | leancrypto | IPP-Crypto |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Geração de chaves (keygen) | — | — | — | — | — | — | — |
| Encapsulamento (encaps) | — | — | — | — | — | — | — |
| Desencapsulamento (decaps) | — | — | — | — | — | — | — |

ML-DSA:

| Operação | mldsa-native | OpenSSL | wolfSSL | BoringSSL | Botan | leancrypto | IPP-Crypto |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Geração de chaves (keygen) | — | — | — | — | — | — | — |
| Assinatura (sign) | — | — | — | — | — | — | — |
| Verificação (verify) | — | — | — | — | — | — | — |

---

## Estrutura do repositório

O repositório é composto por este documento geral e por uma ficha para cada biblioteca analisada.

```
.
├── README.md
└── Bibliotecas/
    ├── BoringSSL.md
    ├── Botan.md
    ├── IPP-Crypto.md
    ├── leancrypto.md
    ├── mlkem-native;mldsa-native.md
    ├── OpenSSL.md
    └── wolfSSL.md
```

Todas as fichas seguem a mesma estrutura, o que permite comparar um mesmo aspecto entre bibliotecas:

1. Identificação
2. Cobertura funcional
3. Implementação por arquitetura
4. Memória e tamanho
5. Segurança
6. Conformidade e certificação
7. Desempenho
8. Pontos positivos
9. Pontos negativos
10. Uso no canal clássico do QKD
11. Fontes

---

## Referências

As referências abaixo são de caráter geral. As fontes específicas de cada biblioteca estão listadas ao final da respectiva ficha.

Normas e ferramentas do NIST:

- [FIPS 203](https://csrc.nist.gov/pubs/fips/203/final): Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM)
- [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final): Module-Lattice-Based Digital Signature Standard (ML-DSA)
- [FIPS 202](https://csrc.nist.gov/pubs/fips/202/final): SHA-3 Standard (SHA-3 e SHAKE)
- [NIST SP 800-227](https://csrc.nist.gov/pubs/sp/800/227/final): recomendações para uso de KEMs (setembro de 2025)
- [Servidor ACVP](https://github.com/usnistgov/ACVP-Server): vetores de teste usados na validação de algoritmos

Canal clássico do QKD:

- Wang et al., "Experimental Authentication of QKD with Post-quantum Cryptography", npj Quantum Information, 2021. Autenticação do canal clássico com assinatura pós-quântica e PKI; a assinatura levou menos de 1 ms em um ciclo de 1 s, sem alteração na taxa de chave. [arXiv 2009.04662](https://arxiv.org/abs/2009.04662)
- Giestinhas e Spiller, "On Post-Quantum Cryptography Authentication for QKD". Propõe os protocolos "assinatura seguida de MAC" e "KEM seguido de MAC", que correspondem aos papéis do ML-DSA e do ML-KEM. [arXiv 2507.21325](https://arxiv.org/abs/2507.21325)
- ANSSI, BSI, NLNCSA e agência sueca, "Position Paper on Quantum Key Distribution". Discute os limites do QKD e a necessidade de autenticar o canal clássico. [cyber.gouv.fr](https://cyber.gouv.fr/en/publications/jointly-led-international-publications/position-paper-on-quantum-key-distribution/)
