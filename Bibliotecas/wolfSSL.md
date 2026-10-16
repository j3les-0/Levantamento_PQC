
---
LINKS:
https://www.wolfssl.com/
https://github.com/wolfSSL/wolfssl

---
# wolfSSL (wolfCrypt)

Biblioteca TLS/DTLS voltada a embarcados e tempo real; a camada criptográfica é o wolfCrypt. Tem ML-KEM, ML-DSA, SLH-DSA, LMS/HSS e XMSS/XMSS^MT.

Última revisão: 16/10/2026.
## 1. Identificação

|Item|Valor|
|---|---|
|Padrões|FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA), LMS/HSS (RFC 8554), XMSS (RFC 8391)|
|Linguagem|C (+ assembly gerado por scripts Ruby para x86_64, ARM e RISC-V)|
|Origem do código|Implementação própria da wolfSSL Inc. O ML-KEM partiu do pacote da 3ª rodada do NIST e foi atualizado para o FIPS 203. O "Dilithium" foi renomeado para ML-DSA na 5.9.2, mantendo o shim `dilithium.h`|
|Mantenedor / governança|wolfSSL Inc. (empresa; desenvolvimento aberto no GitHub)|
|Versão analisada|v5.9.4-stable (25/09/2026); master `0dbc94b55` (13/10/2026)|
|Licença|GPLv3 (com exceção que permite GPLv2 para softwares listados em `LICENSING`) ou licença comercial|
|Integração em protocolos|TLS 1.3 com grupos híbridos (`X25519MLKEM768` etc.); ML-DSA nos sigalgs de (D)TLS 1.3; certificados X.509 com ML-DSA e ML-KEM; KEX híbrido em SSHv2 (wolfSSH)|

## 2. Cobertura funcional

|Característica|ML-KEM|ML-DSA|
|---|---|---|
|Níveis de parâmetro|512 / 768 / 1024 (cada um pode ser desligado: `WOLFSSL_NO_ML_KEM_*`)|44 / 65 / 87 (`WOLFSSL_NO_ML_DSA_*`)|
|Variante pré-hash (HashML-DSA)|Não se aplica|Sim (`wc_MlDsaKey_SignCtxHash` / `VerifyCtxHash`)|
|External mu|Não se aplica|Sim (`wc_MlDsaKey_SignMuWithSeed` / `VerifyMu`)|
|Determinística / aleatorizada|Aleatorizada; `EncapsulateWithRandom` para testes|Hedged por padrão; determinística com seed (`SignCtxWithSeed`); contexto obrigatório por padrão|
|Formatos de chave|Seed e expandida; DER/PKCS#8 e X.509|`MakeKeyFromSeed`; DER/PKCS#8 e X.509|
|Outros recursos|Remoção de operações (`NO_MAKE_KEY`, `NO_ENCAPSULATE`, `NO_DECAPSULATE`); callbacks para hardware (`cryptocb`)|Build só de verificação (`--enable-mldsa=verify-only`); chaves em dispositivo via `cryptocb`|

## 3. Implementação por arquitetura

|Arquitetura|Extensão|ML-KEM|ML-DSA|
|---|---|---|---|
|x86_64|AVX2|●|●|
|x86_64|AVX-512|●|●|
|AArch64|NEON|●|○|
|AArch64|SVE/SVE2|○|○|
|AArch64|SHA3 (Keccak em hardware)|●|●|
|Arm 32 bits / Cortex-M|Escalar (DSP)|●|◐|
|Armv8.1-M|Helium/MVE|○|○|
|RISC-V|RVV|◐|◐|
|RISC-V|Escalar (RV32/RV64)|○|○|
|POWER|VSX (ppc64le)|◐|◐|
|s390x|Vector / CPACF|○|○|

Legenda:

| Marcador | Significado |
| --- | --- |
| ● | As operações principais do algoritmo têm código otimizado para a extensão |
| ◐ | Otimização parcial: só algumas operações, só o Keccak, ou só em versão não lançada ou experimental (detalhes nas notas) |
| ○ | Sem código específico (C portátil) ou arquitetura sem suporte |

Notas por operação:
- **x86_64 AVX2:**
  - ML-KEM: keygen, encaps e decaps inteiros em assembly (`mlkem_*_avx2`).
  - ML-DSA: NTT, redução, amostragem e empacotamento (`wc_mldsa_*_avx2`).
  - Keccak em AVX2.
- **x86_64 AVX-512:**
  - ML-KEM: keygen, encaps e decaps em AVX-512, com variantes VBMI/VBMI2 para compressão, empacotamento e rejeição.
  - ML-DSA: NTT e NTT inversa, multiplicação, decomposição, dicas, codificação e amostragem (`wc_mldsa_*_avx512`, algumas em VBMI).
  - Keccak com 8 instâncias paralelas (`sha3_blocksx8_avx512`).
  - A escolha AVX2 ou AVX-512 é feita em tempo de execução.
- **AArch64:**
  - ML-KEM tem NTT, basemul, CBD e redução em NEON (`armv8-mlkem-asm`). O ML-DSA não tem assembly e só aproveita o Keccak.
  - O Keccak (`armv8-sha3-asm`) usa as instruções SHA3 (`EOR3`, `RAX1`, `XAR`, `BCAX`).
- **Arm 32 bits / Cortex-M (ML-KEM):** assembly escalar com instruções DSP (`smulbb`, `smlabb`) para Armv7-A/Armv8-A32 (`armv8-32-mlkem-asm`) e Thumb-2 para Cortex-M (`thumb2-mlkem-asm`). O Keccak também tem assembly para Arm 32 bits e Thumb-2, e é o único ganho do ML-DSA nesses alvos.
- **RISC-V:** só o Keccak é otimizado, e só na versão vetorial (RVV, com Zvkb/Zvbb opcionais). A aritmética de ML-KEM e ML-DSA fica em C. Não há Keccak escalar em assembly.
- **POWER:** só o Keccak (`ppc64-sha3-asm`, `ppc32-sha3-asm`).
- **Hardware externo:** há builds `WOLF_CRYPTO_CB_ONLY_MLKEM` e `WOLF_CRYPTO_CB_ONLY_MLDSA`, que removem a matemática em software e delegam a um acelerador ou HSM.

## 4. Memória e tamanho

|Item|Valor|
|---|---|
|Uso de heap|Opcional. Builds sem malloc para ML-KEM (desde a 5.8.0) e para a verificação ML-DSA (`WOLFSSL_MLDSA_VERIFY_NO_MALLOC`); alocador estático global (`--enable-staticmemory`)|
|Opções de redução de memória|ML-KEM: `MAKEKEY_SMALL_MEM`, `ENCAPSULATE_SMALL_MEM`, `CACHE_A`, `DYNAMIC_KEYS`. ML-DSA: `SMALL`, `SIGN_SMALL_MEM`, `SIGN_SMALLEST_MEM`, `VERIFY_SMALL_MEM`, `VERIFY_SMALLEST_MEM`, `MAKE_KEY_SMALL_MEM`, `SMALL_MEM_POLY64`, `NO_LARGE_CODE`, `VERIFY_ONLY`, `DYNAMIC_KEYS`|
|Memória por operação documentada|Não publicada; pode ser obtida com `--enable-stacksize` / `--enable-trackmemory`|
|Ambientes|SO completos, RTOS e bare-metal; suporte a alvos com `CHAR_BIT != 8` (TI C2000)|

## 5. Segurança

|Item|Valor|
|---|---|
|Verificação formal|Nenhuma|
|Tempo constante|Código escrito para tempo constante; não há prova nem ferramenta de verificação documentada para PQC|
|Testes|KATs, vetores ACVP v7.0.0 (inclusive external mu e determinístico), testes de API extensos (`tests/api/test_mlkem.c`, `test_mldsa.c`), CI com muitas combinações de build|
|Vulnerabilidades PQC recentes|**CVE-2026-10097 (High):** `mlkem_cmp_avx2` comparava só 1536 dos 1568 bytes do texto cifrado na recriptografia FO do ML-KEM-1024, o que permitia recuperar a chave privada com cerca de 350 textos cifrados escolhidos; corrigida na 5.9.2. Também: double-free no keyshare híbrido truncado (corrigido na 5.9.1) e correções de limites e overflow no ML-DSA|
|Fora de escopo|Potência/EM e falhas não são tratados no código genérico|

## 6. Conformidade e certificação

|Item|Valor|
|---|---|
|Conformidade FIPS 203/204|Testada com vetores ACVP|
|Certificados CMVP ativos|#4718 (wolfCrypt v5.2.1, até 17/07/2030), #5041 (v5.2.0.1, validado em 18/07/2025) e rebrands #5107 e #5199. Nenhum lista ML-KEM, ML-DSA, SLH-DSA ou LMS|
|CMVP com ML-KEM/ML-DSA|Em processo: o wolfCrypt v7.0.0 (ML-KEM, ML-DSA, SLH-DSA e LMS/XMSS só verificação) está em auditoria final de conformidade; a empresa anunciou em 18/02/2026 que a submissão estava "in process"|
|Outras submissões|v5.2.3 e v6.0.0 submetidas (sem PQC); v5.2.4 (módulo de kernel) em testes|

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

- A cobertura mais ampla em x86_64: AVX2 e AVX-512 completos para os dois algoritmos, inclusive Keccak x8.
- ML-KEM em assembly também para AArch64, Arm 32 bits e Cortex-M.
- API completa: HashML-DSA, external mu, seeds e verificação isolada.
- O melhor suporte a embarcados do levantamento: build sem malloc, vários modos de memória, RTOS e alvos DSP.
- Integração com TLS, DTLS e SSH, e delegação a hardware via `cryptocb`.
- Certificação FIPS com PQC em andamento.

## 9. Pontos negativos

- Licença GPLv3 ou comercial, o que restringe o uso em produto fechado.
- Sem verificação formal.
- ML-DSA sem assembly fora de x86_64.
- RISC-V e POWER só com Keccak otimizado.
- Histórico recente de bug grave no caminho otimizado (CVE-2026-10097).
- Certificado FIPS com PQC ainda não emitido.

## 10. Uso no canal clássico do QKD

- **Nós embarcados com TLS/DTLS:** é a escolha mais forte quando os nós do QKD são embarcados (MCU ou SoC com RTOS) e o canal clássico usa TLS ou DTLS 1.3. Pilha completa, modos de memória reduzida e aceleração por hardware via `cryptocb`.
- **Servidores x86:** o AVX-512 nos dois algoritmos dá vantagem quando há muitos enlaces.
- **Licença:** o uso depende de aceitar GPLv3 ou de contratar licença comercial.
- **Risco de bugs:** a falta de verificação formal, junto com o CVE recente, pede validação independente da implementação antes de uso em produção.

## 11. Fontes

- Código clonado: `wolfssl` master `0dbc94b55` (`wolfcrypt/src/wc_mlkem*.c`, `wc_mldsa.c`, `*_asm.S`, `port/arm`, `port/riscv64`, `port/ppc*`, `wolfssl/wolfcrypt/wc_mldsa.h`, `ChangeLog.md`, `LICENSING`, `tests/api/test_ml*.c`)
- Página PQC: https://www.wolfssl.com/products/wolfcrypt-post-quantum/
- Página FIPS: https://www.wolfssl.com/license/fips/
- Comunicado "wolfCrypt FIPS 140-3 with Post-Quantum Cryptography Available Now": https://www.prweb.com/releases/wolfcrypt-fips-140-3-with-post-quantum-cryptography-available-now-302691556.html
- Certificado #5041: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/5041
- Benchmarks em STM32: https://www.wolfssl.com/updated-post-quantum-benchmarks-for-ml-kem-and-ml-dsa-on-stm32/
- Guia de migração Dilithium → ML-DSA: `doc/dilithium-to-mldsa-migration.md` (no repositório)

---
