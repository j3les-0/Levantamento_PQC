
---
LINKS:
https://www.openssl.org/
https://github.com/openssl/openssl

---
# OpenSSL

Biblioteca de uso geral (libcrypto + libssl, com TLS, DTLS e QUIC). Tem ML-KEM, ML-DSA e SLH-DSA nativos desde a 3.5.0 (abril/2025), nos providers `default` e `fips`.



## 1. Identificação

|Item|Valor|
|---|---|
|Padrões|FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA); LMS (verificação)|
|Linguagem|C (+ assembly gerado por perlasm)|
|Origem do código|ML-KEM derivado do código do BoringSSL (contribuído sob CCLA, com API refatorada; autor principal Viktor Dukhovni). ML-DSA é implementação própria (autor principal Shane Lontis)|
|Mantenedor / governança|OpenSSL Foundation e OpenSSL Corporation|
|Versão analisada|master `b86b1098fd` (13/10/2026, 4.2.0-dev). Releases: 3.5 (LTS, suporte até 08/04/2030), 3.6 (até 01/11/2026), 4.0.3 (29/09/2026), 4.1.0-beta2 (09/10/2026)|
|Licença|Apache-2.0|
|Integração em protocolos|TLS 1.3 com ML-KEM puro e híbridos (`X25519MLKEM768`, padrão desde a 3.5; `SecP256r1MLKEM768`; `SecP384r1MLKEM1024`); ML-DSA nos algoritmos de assinatura TLS; X.509; CMS com `KEMRecipientInfo` (RFC 9629)|

## 2. Cobertura funcional

|Característica|ML-KEM|ML-DSA|
|---|---|---|
|Níveis de parâmetro|512 / 768 / 1024|44 / 65 / 87|
|Variante pré-hash (HashML-DSA)|Não se aplica|**Não** (só ML-DSA "pure")|
|External mu|Não se aplica|Sim (parâmetro `mu`; digest `ML-DSA-MU` desde a 4.0)|
|Determinística / aleatorizada|Aleatorizada; `ikme` só para testes|Hedged por padrão; determinística com o parâmetro `deterministic`; contexto suportado|
|Formatos de chave|Seed, expandida ou ambas (`seed-priv`, `seed-only`, `priv-only`, `bare-seed`, `oqskeypair` etc.), configuráveis por chave|Seed, expandida ou ambas, configuráveis por chave|
|Outros recursos|Híbridos para TLS (MLX-KEM)|SLH-DSA disponível na mesma API EVP|

## 3. Implementação por arquitetura



|Arquitetura|Extensão|ML-KEM|ML-DSA|
|---|---|---|---|
|x86_64|AVX2|○|◐|
|x86_64|AVX-512|○|◐|
|AArch64|NEON|○|○|
|AArch64|SVE/SVE2|○|○|
|AArch64|SHA3 (Keccak em hardware)|●|●|
|Arm 32 bits / Cortex-M|Escalar (DSP)|○|○|
|Armv8.1-M|Helium/MVE|○|○|
|RISC-V|RVV|○|○|
|RISC-V|Escalar (RV32/RV64)|○|○|
|POWER|VSX (ppc64le)|◐|◐|
|s390x|Vector / CPACF|◐|◐|

Legenda:

| Marcador | Significado |
| --- | --- |
| ● | As operações principais do algoritmo têm código otimizado para a extensão |
| ◐ | Otimização parcial: só algumas operações, só o Keccak, ou só em versão não lançada ou experimental (detalhes nas notas) |
| ○ | Sem código específico (C portátil) ou arquitetura sem suporte |



Notas por operação:
- **Até a 4.0.x**, a aritmética de ML-KEM e ML-DSA é toda em C portátil em todas as arquiteturas. As otimizações específicas de PQC só começam na 4.1.
- **x86_64:**
  - O Keccak usa o assembly escalar `keccak1600-x86_64` nas duas.
  - **AVX2:** só a NTT e a NTT inversa do ML-DSA (`ml_dsa_ntt-x86_64.pl`, a partir da 4.1).
  - **AVX-512:** só o SHAKE x4 em AVX-512VL, usado na amostragem do ML-DSA (a partir da 4.1).
- **AArch64:** sem código vetorial para ML-KEM ou ML-DSA. O Keccak (`keccak1600-armv8`) usa as instruções SHA3 quando a CPU as tem; esse é o único ganho nessa arquitetura.
- **POWER (ppc64le):** NTT e NTT inversa em assembly para ML-KEM e ML-DSA, a partir da 4.1. O Keccak tem assembly `keccak1600-ppc64`.
- **s390x:**
  - NTT vetorial do ML-DSA a partir da 4.1; a do ML-KEM só na master (4.2-dev).
  - SHA-3 por instrução CPACF (KIMD/KLMD) nas duas.
- **Sem otimização:** RISC-V e Cortex-M não têm código específico, e não há Keccak em assembly para RISC-V. O OpenSSL não tem como alvo microcontroladores.

## 4. Memória e tamanho

|Item|Valor|
|---|---|
|Uso de heap|Sim. As chaves são alocadas com `OPENSSL_malloc`/`zalloc`, e a camada EVP/providers também aloca contextos|
|Opções de redução de memória|Nenhuma específica para PQC. Algoritmos podem ser removidos no `Configure` (`no-ml-kem`, `no-ml-dsa` etc.)|
|Memória por operação documentada|Não|
|Ambientes|Sistemas operacionais completos (POSIX, Windows). Não é alvo de bare-metal; `libcrypto` inclui TLS, X.509, ASN.1 e todos os algoritmos clássicos|

## 5. Segurança

|Item|Valor|
|---|---|
|Verificação formal|Nenhuma|
|Tempo constante|Primitivas `constant_time_*` e reduções de Barrett sem divisão. A instrumentação valgrind (`enable-ct-validation`) marca as entradas secretas de keygen, encaps e decaps do ML-KEM, numa abordagem inspirada no BoringSSL|
|Testes|KAT, vetores ACVP do NIST, Wycheproof (ML-DSA), testes de interoperabilidade TLS|
|Autotestes|PCT na importação de chaves no modo FIPS (ML-KEM com `import_pct_type`; SLH-DSA também)|
|Fora de escopo|Canais laterais de potência/EM e injeção de falhas|

## 6. Conformidade e certificação

|Item|Valor|
|---|---|
|Conformidade FIPS 203/204|Testada com vetores ACVP e Wycheproof|
|Certificados CMVP ativos|#4985 (FIPS 140-3, provider 3.1.2, sunset 10/03/2030); #4282 e #4811 (FIPS 140-2, provider 3.0.8/3.0.9, sunset 21/09/2026). Nenhum inclui PQC|
|CMVP com ML-KEM/ML-DSA|Em processo: a 3.5.4 foi submetida em 09/10/2025 (laboratório Lightship Security); segundo o projeto, a 3.5.8 está em revisão no CMVP|
|Indicador FIPS|`fips-indicator` para ML-KEM, ML-DSA e SLH-DSA; entropia de teste torna a operação não aprovada|

Alternativas comerciais derivadas do OpenSSL, com validação própria em andamento: SafeLogic CryptoComply e o provider FIPS da Chainguard para OpenSSL 3.6.

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

- Integração completa em protocolo: TLS 1.3 com troca híbrida por padrão, X.509, CMS e QUIC.
- API EVP madura e conhecida, com controle fino dos formatos de chave.
- Versão LTS (3.5) com suporte até 2030.
- External mu e assinatura determinística.
- Testes com vetores ACVP e Wycheproof.
- Submissão FIPS com PQC já em revisão no CMVP.
- Licença Apache-2.0.

## 9. Pontos negativos

- Sem HashML-DSA.
- Sem verificação formal.
- ML-KEM sem nenhuma otimização vetorial em x86 ou ARM, em todas as versões.
- ML-DSA otimizado apenas a partir da 4.1, e só em x86_64, POWER e s390x.
- Nada para AArch64 além do Keccak, e nada para RISC-V.
- Usa heap e é pesado para sistemas embarcados.
- Certificado CMVP com PQC ainda não emitido.

## 10. Uso no canal clássico do QKD

- **Canal em TLS 1.3:** é a opção mais direta. Autenticação mútua com certificados ML-DSA e troca híbrida `X25519MLKEM768` já vêm prontas no `libssl`, sem código de protocolo próprio.
- **Ambientes adequados:** servidores ou nós com sistema operacional completo, onde tamanho de binário e uso de heap não importam. A LTS 3.5 dá estabilidade de longo prazo para equipamentos de infraestrutura.
- **Restrições:**
  - a ausência de HashML-DSA limita esquemas que assinam resumos calculados fora do módulo;
  - a falta de otimização em ARM pesa se os nós forem SoC ARM;
  - se for exigida certificação FIPS, ainda é preciso aguardar o certificado com PQC.

## 11. Fontes

- Código clonado: `openssl` master `b86b1098fd` (`crypto/ml_kem`, `crypto/ml_dsa`, `crypto/sha/build.info`, `CHANGES.md`, `doc/man7/EVP_*-ML-*.pod`, `doc/man7/fips_module.pod`, `test/recipes/30-test_evp_data`)
- Commits de origem: `42436eb53e` (KATs do BoringSSL sob CCLA), `95d764a044` (instrumentação valgrind), `d2136d9e73` (ML-KEM multinível)
- Política de versões e LTS: https://openssl-library.org/policies/releasestrat/
- Anúncio da 4.1 alpha: https://openssl-library.org/post/2026-09-09-openssl-4.1-alpha/
- Documentação ML-DSA: https://docs.openssl.org/master/man7/EVP_SIGNATURE-ML-DSA/
- Documentação ML-KEM: https://docs.openssl.org/master/man7/EVP_PKEY-ML-KEM/
- Provider FIPS e certificados: https://openssl-library.org/news/fips-cve/
- Submissão da 3.5.4: https://openssl-corporation.org/blog/openssl-354-fips-submission.html
- Security Policy #4985: https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp4985.pdf

---
