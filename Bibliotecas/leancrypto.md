
---
LINKS:
https://leancrypto.org/
https://github.com/smuellerDD/leancrypto

---
# leancrypto

Biblioteca C "enxuta", feita para rodar em qualquer ambiente: espaço de usuário POSIX/Windows, kernel Linux (módulo ou estático), EFI/secure boot e bare-metal (*freestanding*). O README diz que ela contém "exclusivamente algoritmos resistentes a PQC", mas também inclui curvas clássicas (X25519/X448, Ed25519/Ed448), usadas sobretudo nos esquemas híbridos. Oferece SHA3 e SHAKE isolados, que podem ser extraídos como componentes independentes. Autor: Stephan Müller.



## 1. Identificação

|Item|Valor|
|---|---|
|Padrões|FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA); também BIKE, HQC e sntrup (761 a 1277)|
|Linguagem|C (+ assembly); bindings Rust (inclusive provider rustls), Python e PKCS#11|
|Origem do código|Derivado das referências do CRYSTALS (Kyber e Dilithium) e de trabalhos otimizados: ML-KEM ARMv8 de `psanal2018/kyber-arm64`, ML-DSA ARMv8 do PQClean, ARMv7/Cortex-M de `dilithium-cortexm`, RISC-V do PQRV (NTT Plantard)|
|Mantenedor / governança|Projeto individual (Stephan Müller), código aberto e com CI extensa|
|Versão analisada|v1.9.1 (28/09/2026); master `e491f454` (11/10/2026, 1.9.2-prerelease)|
|Licença|BSD-3-Clause ou GPLv2, à escolha do usuário|
|Integração em protocolos|Nenhuma pilha TLS própria (há provider para rustls); X.509 e PKCS#7 com ML-DSA (assinatura de módulos do kernel)|

## 2. Cobertura funcional

|Característica|ML-KEM|ML-DSA|
|---|---|---|
|Níveis de parâmetro|512 / 768 / 1024|44 / 65 / 87|
|Variante pré-hash (HashML-DSA)|Não se aplica|Sim|
|External mu|Não se aplica|Sim|
|Determinística / aleatorizada|Aleatorizada|Hedged por padrão; determinística disponível; contexto suportado (`sign_ctx`)|
|Formatos de chave|Formato FIPS 203|Formato FIPS 204|
|Outros recursos|KEM híbrido com X25519/X448; modos KEX (troca autenticada) e IES (criptografia integrada)|Assinatura híbrida com Ed25519/Ed448|

## 3. Implementação por arquitetura



| Arquitetura            | Extensão                  | ML-KEM | ML-DSA |
| ---------------------- | ------------------------- | ------ | ------ |
| x86_64                 | AVX2                      | ●     | ●     |
| x86_64                 | AVX-512                   | ◐     | ◐     |
| AArch64                | NEON                      | ●     | ●     |
| AArch64                | SVE/SVE2                  | ○      | ○      |
| AArch64                | SHA3 (Keccak em hardware) | ●     | ●     |
| Arm 32 bits / Cortex-M | Escalar (DSP)             | ●     | ●     |
| Armv8.1-M              | Helium/MVE                | ○      | ○      |
| RISC-V                 | RVV                       | ●     | ●     |
| RISC-V                 | Escalar (RV32/RV64)       | ●     | ●     |
| POWER                  | VSX (ppc64le)             | ○      | ○      |
| s390x                  | Vector / CPACF            | ○      | ○      |

Legenda:

| Marcador | Significado |
| --- | --- |
| ● | As operações principais do algoritmo têm código otimizado para a extensão |
| ◐ | Otimização parcial: só algumas operações, só o Keccak, ou só em versão não lançada ou experimental (detalhes nas notas) |
| ○ | Sem código específico (C portátil) ou arquitetura sem suporte |



Notas por operação:
- **x86_64 AVX2:**
  - ML-KEM: NTT, basemul, CBD, rejeição e shuffle.
  - ML-DSA: NTT, multiplicação pontual e polinômios.
  - Keccak em AVX2, inclusive com 4 instâncias paralelas.
- **x86_64 AVX-512:** só o Keccak; a aritmética usa o caminho AVX2.
- **AArch64:**
  - ML-KEM: NTT, basemul, CBD e redução em NEON.
  - ML-DSA: NTT e NTT inversa em NEON.
  - Keccak em NEON, com instruções SHA3 (`armv8a-ce`) e em 2 instâncias com SHA3 (`keccakx2_armce`).
- **Arm 32 bits / Cortex-M:** NTT e NTT inversa em assembly ARMv7 (Thumb-2/DSP) para os dois algoritmos, chamadas a partir do código C. Em ARMv7-A, o Keccak também tem versão NEON.
- **RISC-V RVV:** ML-KEM com implementações separadas para VLEN 128 e 256, escolhidas automaticamente em tempo de execução; ML-DSA também em RVV.
- **RISC-V escalar:** NTT em rv64im (Plantard "dual-issue") nos dois algoritmos; Keccak em rv64im/rv64imb e em rv32.
- **Seleção de implementação:** um seletor em tempo de execução escolhe a mais rápida que a CPU suporta.

## 4. Memória e tamanho

|Item|Valor|
|---|---|
|Uso de heap|Opcional. Todos os algoritmos podem ser alocados na pilha (macros `*_CTX_ON_STACK`); alocação em heap também é suportada|
|Opções de redução de memória|Opção `small_stack` (uso de pilha ≤ 2048 bytes, com CI dedicada); desativação de qualquer algoritmo no build; *dead-code stripping* no link estático|
|Memória por operação documentada|Não|
|Ambientes|Espaço de usuário POSIX e Windows, kernel Linux, EFI e freestanding/bare-metal (por exemplo, secure boot do i-RISCV)|

## 5. Segurança

|Item|Valor|
|---|---|
|Verificação formal|Nenhuma|
|Tempo constante|Análise dinâmica com valgrind ("timecop") na CI, para encontrar caminhos dependentes de segredo|
|Testes|Vetores ACVP (via `acvpproxy` do próprio autor), testador Wycheproof em Rust, UBSAN, testes no kernel, Windows, BSDs e Solaris|
|Autotestes|CAST por algoritmo antes do primeiro uso; PCT; modo degradado (só o algoritmo que falhou é desativado)|
|Fora de escopo|Potência/EM e falhas|

## 6. Conformidade e certificação

|Item|Valor|
|---|---|
|Conformidade FIPS 203/204|Testada com vetores ACVP|
|CAVP|Sim. **Primeiros certificados CAVP do mundo para ML-KEM e ML-DSA** (v1.0.0, ago/2024), renovados até a v1.6.0 (out/2025). Nas versões 0.x, cada implementação tinha certificado próprio (C, AVX2, AVX-512, 4-way SIMD, RISC-V, ARMv8, ARMv8 CE, ARMv8 2-way, ARMv7 NEON)|
|CMVP (FIPS 140-3)|Não há certificado de módulo|
|Estrutura FIPS|Sempre compilada como módulo FIPS: `leancrypto-fips.so` com teste de integridade SHA3-256, indicador de serviço, autotestes e modo degradado. A validação do módulo fica a cargo de quem o incorpora|

## 7. Desempenho

|Algoritmo|keygen|encaps / sign|decaps / verify|
|---|---|---|---|
|ML-KEM-512||||
|ML-KEM-768||||
|ML-KEM-1024||||
|ML-DSA-44||||
|ML-DSA-65||||
|ML-DSA-87||||

## 8. Pontos positivos

- A cobertura de arquiteturas mais ampla para ML-DSA do levantamento: AVX2, NEON, ARMv7/Cortex-M, RVV e rv64im.
- RISC-V vetorial e escalar para os dois algoritmos.
- API completa: HashML-DSA, external mu, híbridos, KEX e IES.
- Uso só de pilha possível, com modo de pilha ≤ 2 KB.
- Roda no kernel, em EFI e em bare-metal.
- CAVP para ML-KEM e ML-DSA desde o primeiro dia.
- Licença BSD-3 ou GPLv2.

## 9. Pontos negativos

- Projeto com um mantenedor principal (risco de continuidade).
- Sem AVX-512 nem SVE na aritmética.
- Sem verificação formal.
- Sem certificado CMVP de módulo.
- Sem pilha TLS própria.

## 10. Uso no canal clássico do QKD

- **Firmware próprio em ARM ou RISC-V:** é a opção mais atraente para equipamentos QKD com SoC ARM ou RISC-V e firmware próprio (inclusive kernel e boot), que precisam de ML-DSA otimizado sem heap.
- **Protocolo próprio:** os modos KEX e IES sobre ML-KEM e as assinaturas híbridas permitem montar a autenticação do canal clássico sem uma pilha TLS.
- **Base para certificação:** a estrutura de módulo FIPS e o CAVP facilitam uma certificação futura do equipamento.
- **Quando o canal usa TLS:** é preciso combiná-la com outro componente de protocolo.

## 11. Fontes

- Código clonado: `leancrypto` master `e491f454` (`ml-kem/src/{avx2,armv8,armv7,riscv64}`, `ml-dsa/src/*`, `hash/src/asm/*`, `ml-dsa/api/lc_dilithium.h`, `README.md`, `README.fips.md`, `LICENSE*`, `meson_options.txt`, `CHANGES.md`)
- Site: https://leancrypto.org/
- Certificados CAVP: https://leancrypto.org/leancrypto/cavp_certificates/index.html
- Suporte FIPS 140: https://leancrypto.org/leancrypto/fips_support/index.html
- Algoritmos: https://leancrypto.org/leancrypto/cryptographic_algorithms/index.html
- Definição ACVP: https://github.com/smuellerDD/acvpproxy/blob/master/lib/module_implementations/definition_impl_leancrypto.c
- Releases: https://github.com/smuellerDD/leancrypto/releases

---
