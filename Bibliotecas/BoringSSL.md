
---
LINKS:
https://boringssl.googlesource.com/boringssl
https://github.com/google/boringssl

---
# BoringSSL

Fork do OpenSSL mantido pelo Google para uso próprio (Chrome, Android e serviços internos). Não promete API estável; o projeto recomenda fixar um commit. Oferece ML-KEM, ML-DSA, SLH-DSA e X-Wing; a parte FIPS se chama **BoringCrypto** (BCM).



## 1. Identificação

|Item|Valor|
|---|---|
|Padrões|FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA); X-Wing (KEM híbrido X25519 + ML-KEM-768)|
|Linguagem|C++|
|Origem do código|Implementação própria do Google: ML-KEM adicionado em 21/08/2024; ML-DSA, Keccak e SLH-DSA movidos para o módulo FIPS em dez/2024. O ML-KEM do OpenSSL deriva deste código|
|Mantenedor / governança|Google|
|Versão analisada|tag `0.20260929.0` (29/09/2026); main `5b83e5d1a` (11/10/2026). Versionamento por data, sem releases LTS|
|Licença|Apache-2.0 (arquivo `LICENSE` atual, com seções de terceiros)|
|Integração em protocolos|TLS 1.3 com `X25519MLKEM768` e `MLKEM1024`; pilha TLS do Chrome e do Android. A API de ML-DSA é de baixo nível|

## 2. Cobertura funcional

|Característica|ML-KEM|ML-DSA|
|---|---|---|
|Níveis de parâmetro|**Só 768 / 1024** (sem 512 na API pública e no BCM)|44 / 65 / 87|
|Variante pré-hash (HashML-DSA)|Não se aplica|**Não.** As funções `MLDSA*_prehash_*` calculam o μ em streaming; não são HashML-DSA|
|External mu|Não se aplica|Sim (`prehash_init/update/finalize` + `sign_message_representative` / `verify_message_representative`)|
|Determinística / aleatorizada|Aleatorizada|Só aleatorizada na API pública; a determinística existe apenas internamente no BCM, para ACVP|
|Formatos de chave|Seed opcional no keygen; estruturas de chave de tamanho fixo|Keygen a partir de seed (`private_key_from_seed`), `public_from_private`|
|Outros recursos|X-Wing|Contexto de até 255 bytes|

## 3. Implementação por arquitetura


|Arquitetura|Extensão|ML-KEM|ML-DSA|
|---|---|---|---|
|x86_64|AVX2|◐|◐|
|x86_64|AVX-512|○|○|
|AArch64|NEON|◐|◐|
|AArch64|SVE/SVE2|○|○|
|AArch64|SHA3 (Keccak em hardware)|○|○|
|Arm 32 bits / Cortex-M|Escalar (DSP)|○|○|
|Armv8.1-M|Helium/MVE|○|○|
|RISC-V|RVV|○|○|
|RISC-V|Escalar (RV32/RV64)|○|○|
|POWER|VSX (ppc64le)|○|○|
|s390x|Vector / CPACF|○|○|

Legenda:

| Marcador | Significado |
| --- | --- |
| ● | As operações principais do algoritmo têm código otimizado para a extensão |
| ◐ | Otimização parcial: só algumas operações, só o Keccak, ou só em versão não lançada ou experimental (detalhes nas notas) |
| ○ | Sem código específico (C portátil) ou arquitetura sem suporte |


Notas por operação:
- Não há assembly escrito à mão para ML-KEM, ML-DSA ou Keccak. A aritmética (NTT, multiplicação, amostragem) é C++ portátil em todas as arquiteturas.
- **x86_64 AVX2 e AArch64 NEON:** só o Keccak é vetorizado. Desde 08/07/2026 há versões x2/x4 escritas com as *vector extensions* do compilador:
  - em x86_64, compiladas com `target("avx2")` e escolhidas em tempo de execução;
  - em AArch64, traduzidas para NEON.
  - São usadas na expansão da matriz A e na amostragem (x4 no ML-KEM; x2 e x4 no ML-DSA).
- **AArch64 SHA3:** as instruções SHA3 de hardware não são usadas.

## 4. Memória e tamanho

|Item|Valor|
|---|---|
|Uso de heap|ML-KEM: não (estruturas de tamanho fixo). ML-DSA: sim, temporários grandes de keygen, sign e verify alocados com `MakeUnique`|
|Opções de redução de memória|Nenhuma|
|Memória por operação documentada|Só o tamanho das estruturas de chave nos headers|
|Ambientes|SO completos (Linux, Android, Windows, macOS); não é alvo de microcontroladores|

## 5. Segurança

|Item|Valor|
|---|---|
|Verificação formal|Nenhuma declarada para ML-KEM/ML-DSA|
|Tempo constante|Anotações `CONSTTIME_SECRET`/`CONSTTIME_DECLASSIFY` (7 no ML-KEM, 16 no ML-DSA), verificadas com valgrind no build `-DCONSTANT_TIME_VALIDATION=1`|
|Testes|Vetores NIST/ACVP, Wycheproof (ML-KEM e ML-DSA), autoconsistência, `modulewrapper` para ACVP|
|Autotestes FIPS|PCT de ML-KEM, ML-DSA e SLH-DSA no keygen; KATs (`fips_known_values.inc`)|
|Fora de escopo|Potência/EM e falhas|

## 6. Conformidade e certificação

|Item|Valor|
|---|---|
|Conformidade FIPS 203/204|Testada com vetores NIST e Wycheproof|
|CAVP com PQC|Sim: A7700 (BoringCrypto 20251031, 21/11/2025) com ML-KEM, ML-DSA e SLH-DSA|
|Certificados CMVP ativos|#5296 (BoringCrypto 2023042800, validado em 02/06/2026, sunset 26/01/2030) e anteriores (#2964 a #5244). Nenhum inclui PQC; a Security Policy do #5296 não menciona SHA-3|
|CMVP com ML-KEM/ML-DSA|Em revisão no NIST: versão 2025-10-31 (CAVP A7700); também as versões 2025-01-07 (A6838) e 2025-07-28 (A7303)|

O BoringSSL como um todo não é validado, só o BCM. O Google não responde a perguntas sobre FIPS. Há alternativa comercial validada baseada no BoringCrypto (SafeLogic CryptoComply BoringCrypto).

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

- Código compacto, muito revisado e usado em escala global (Chrome, Android).
- API simples, com tipos de tamanho fixo.
- ML-KEM sem heap.
- Wycheproof nos testes e anotações de tempo constante verificadas com valgrind.
- X-Wing e SLH-DSA disponíveis.
- CAVP já emitido para ML-KEM, ML-DSA e SLH-DSA.
- Licença Apache-2.0.

## 9. Pontos negativos

- Sem ML-KEM-512 e sem HashML-DSA.
- Sem assinatura determinística na API pública.
- Sem assembly e sem NTT vetorizada; só o Keccak é paralelizado.
- Sem estabilidade de API nem releases LTS.
- CMVP com PQC ainda não emitido.
- ML-DSA usa heap.

## 10. Uso no canal clássico do QKD

- **Pilhas Google:** encaixa quando o sistema já usa Chromium, gRPC ou Android, que trazem o BoringSSL embutido.
- **Produto próprio:** a falta de API estável e de releases versionadas aumenta o custo de manutenção num equipamento de longa vida útil. O AWS-LC, fork do BoringSSL com mlkem-native e mldsa-native, cumpre o mesmo papel com API estável.
- **Esquemas restritos:** sem ML-KEM-512 e sem HashML-DSA, o BoringSSL não atende esquemas que dependam dessas variantes.

## 11. Fontes

- Código clonado: `boringssl` main `5b83e5d1a` (`crypto/fipsmodule/{mlkem,mldsa,keccak}`, `include/openssl/{mlkem,mldsa,xwing}.h`, `crypto/fipsmodule/FIPS.md`, `third_party/wycheproof_testvectors`, `ssl/ssl_key_share.cc`)
- Commits: `500fa1f9d` ("Add ML-KEM", 21/08/2024); "Implement a vectorized quad Keccak" (08/07/2026)
- Certificado #5296: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/5296
- Security Policy #5296: https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp5296.pdf
- CAVP A7700: https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program/details?validation=40310
- CAVP A6838: https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program/details?product=19570
- Repositório oficial: https://boringssl.googlesource.com/boringssl

---
