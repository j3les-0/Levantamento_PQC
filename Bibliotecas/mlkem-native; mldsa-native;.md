
---
LINKS:
https://github.com/pq-code-package
https://github.com/pq-code-package/mlkem-native
https://github.com/pq-code-package/mldsa-native

---
# PQ Code Package (PQCP): mlkem-native e mldsa-native

Duas bibliotecas irmãs em C90, focadas apenas em ML-KEM (mlkem-native) e ML-DSA (mldsa-native). São mantidas pela Post-Quantum Cryptography Alliance (Linux Foundation) e têm o maior nível de verificação formal entre as bibliotecas do levantamento.


## 1. Identificação

|Item|ML-KEM (mlkem-native)|ML-DSA (mldsa-native)|
|---|---|---|
|Padrões|FIPS 203|FIPS 204|
|Linguagem|C90 (+ `stdint.h`, inteiro de 64 bits) e assembly|C90 e assembly|
|Origem do código|Fork da referência do CRYSTALS-Kyber|Fork da referência do CRYSTALS-Dilithium|
|Mantenedor / governança|Post-Quantum Cryptography Alliance (Linux Foundation)|idem|
|Versão analisada|v2.0.0 (07/08/2026); commit `c2827a8` (12/10/2026). A v1.3 recebe correções até fev/2027|v2.0.0 (06/08/2026, primeira estável); commit `a14b01a` (13/10/2026)|
|Licença|Apache-2.0 OR ISC OR MIT|Apache-2.0 OR ISC OR MIT|
|Integração em protocolos|Nenhuma própria. Usado por AWS-LC (desde a v1.50.0; a `main` importa a v2.0.0), libOQS (padrão desde a 0.13.0), rustls (via AWS-LC) e Pavona|Nenhuma própria. Usado por AWS-LC (a `main` importa a v2.0.0), libOQS, Pavona e CHERIoT-PQC|

Mudanças de API da v2.0.0 do mlkem-native em relação à v1.3:
- o código `MLK_ERR_FAIL` foi dividido em `MLK_ERR_INVALID_PK`, `MLK_ERR_INVALID_SK` e `MLK_ERR_PCT_FAIL`;
- os aliases SUPERCOP `crypto_kem_*` foram removidos;
- a configuração passou a ser feita só pelo arquivo de config;
- algumas flags de capacidade e de backend foram renomeadas.

## 2. Cobertura funcional

|Característica|ML-KEM (mlkem-native)|ML-DSA (mldsa-native)|
|---|---|---|
|Níveis de parâmetro|512 / 768 / 1024 (`MLK_CONFIG_PARAMETER_SET`, em tempo de compilação; build multinível disponível)|44 / 65 / 87 (`MLD_CONFIG_PARAMETER_SET`)|
|Variante pré-hash (HashML-DSA)|Não se aplica|Sim: `signature_pre_hash_internal` (12 hashes, SHA2-224 a SHAKE-256) e API estável `signature_pre_hash_shake256`|
|External mu|Não se aplica|Sim (`signature_extmu` / `verify_extmu`)|
|Determinística / aleatorizada|`keypair_derand` e `enc_derand` para KAT; uso normal aleatorizado|Hedged por padrão; determinística via `signature_internal` com `rnd` zerado|
|Formatos de chave|Formato FIPS 203; validação de chaves (`check_pk`, `check_sk`)|Formato FIPS 204; `pk_from_sk`|
|Outros recursos|PCT opcional no keygen|Assinatura reiniciável (`examples/restartable_sign`); build só determinístico (`MLD_CONFIG_NO_RANDOMIZED_API`); só API central (`MLD_CONFIG_CORE_API_ONLY`)|

## 3. Implementação por arquitetura

|Arquitetura|Extensão|ML-KEM (mlkem-native)|ML-DSA (mldsa-native)|
|---|---|---|---|
|x86_64|AVX2|●|●|
|x86_64|AVX-512|○|○|
|AArch64|NEON|●|●|
|AArch64|SVE/SVE2|○|○|
|AArch64|SHA3 (Keccak em hardware)|●|●|
|Arm 32 bits / Cortex-M|Escalar (DSP)|○|○|
|Armv8.1-M|Helium/MVE|◐|◐|
|RISC-V|RVV|◐|○|
|RISC-V|Escalar (RV32/RV64)|○|●|
|POWER|VSX (ppc64le)|●|○|
|s390x|Vector / CPACF|○|○|

Legenda:

| Marcador | Significado |
| --- | --- |
| ● | As operações principais do algoritmo têm código otimizado para a extensão |
| ◐ | Otimização parcial: só algumas operações, só o Keccak, ou só em versão não lançada ou experimental (detalhes nas notas) |
| ○ | Sem código específico (C portátil) ou arquitetura sem suporte |


Notas por operação:
- **x86_64 AVX2:** a aritmética herda o AVX2 oficial dos times Kyber/Dilithium; o Keccak roda em AVX2 com 4 instâncias paralelas (x4).
- **AArch64:**
  - NTT derivada do trabalho "Neon NTT", com assembly otimizado pelo SLOTHY.
  - Keccak em variantes escalar (`x1_scalar`), com instruções SHA3 (`x1_v84a`, `x2_v84a`) e híbridas escalar + vetorial (`x4_v8a_scalar`, `x4_v8a_v84a_scalar`). A variante com SHA3 é escolhida quando a CPU tem Armv8.4-SHA3.
- **Armv8.1-M MVE:** backend experimental e desligado por padrão nas duas bibliotecas.
- **RISC-V RVV (mlkem-native):** RVV 1.0 em rv64gc. Funciona para qualquer VLEN, mas a NTT e a NTT inversa só são vetorizadas com VLEN = 256; nos outros casos caem para C. O Keccak fica em C.
- **RISC-V escalar (mldsa-native):** backend RV32IM (base + extensão M) com NTT, NTT inversa e multiplicação pontual Montgomery. O perfil `fastmul` é o padrão; há um perfil shift/add (`slowmul`) para núcleos sem multiplicador rápido. O Keccak fica em C.
- **POWER (mlkem-native):** só aritmética em VSX, com Keccak em C. O README raiz cita POWER8+; o do backend cita POWER9+ ou ISA 2.07+, que corresponde ao POWER8.
- **Sem otimização:** não há código AVX-512, SVE ou Cortex-M escalar em nenhuma das duas.
- **Build só em C:** é possível retirar os diretórios `native` ou fornecer backends próprios (aritmética e FIPS 202) pela interface `native/api.h`.

## 4. Memória e tamanho

|Item|ML-KEM (mlkem-native)|ML-DSA (mldsa-native)|
|---|---|---|
|Uso de heap|Não (tudo na pilha)|Não (tudo na pilha)|
|Opções de redução de memória|`MLK_CONFIG_CUSTOM_ALLOC_FREE` (experimental): buffers grandes por alocador próprio; restam até 3 KB de pilha|`MLD_CONFIG_REDUCE_RAM` (troca desempenho por memória) e `MLD_CONFIG_CUSTOM_ALLOC_FREE`|
|Memória por operação documentada|Sim (`MLK_TOTAL_ALLOC_*` em `mlkem_native.h`)|Sim (`MLD_TOTAL_ALLOC_*` em `mldsa_native.h`)|
|Ambientes|Qualquer alvo com compilador C90; sem dependência de SO|idem|


## 5. Segurança

|Item|ML-KEM (mlkem-native)|ML-DSA (mldsa-native)|
|---|---|---|
|Verificação formal (C)|CBMC: ausência de UB, segurança de memória e de tipos e terminação de laços (exceto a amostragem por rejeição) em `mlkem/src/*` e `fips202/*`|CBMC: idem para todo o C do backend em C|
|Verificação formal (assembly)|HOL-Light + s2n-bignum: correção funcional, segurança de memória e tempo constante, no código objeto, em AArch64 e x86_64|Idem em AArch64, x86_64 e RV32IM, com exceção dos samplers (ver abaixo)|
|Outras provas|—|Isabelle/HOL experimental: rotinas de decomposição e núcleo aritmético do Neon NTT|
|Sem prova|Backends RVV, POWER e MVE|Backend MVE|
|Tempo constante|Provado no assembly verificado. O C usa barreiras de valor contra KyberSlash e clangover e assume multiplicação 32×32 em tempo constante. Os testes com valgrind rodam em x86_64 e AArch64, na configuração padrão|Provado no assembly, exceto `rej_uniform` (dados públicos, propositalmente de tempo variável) e `rej_uniform_eta{2,4}` (tempo constante validado com valgrind, ainda sem prova; issue #1160)|
|Testes|KAT, ACVP, Wycheproof, valgrind|ACVP, Wycheproof, valgrind|
|Fora de escopo (`SOUNDNESS.md`)|Potência, EM, canais microarquiteturais e injeção de falhas. Também não há prova de correção funcional nem de tempo constante do C, nem garantia contra remontagem do objeto por terceiros|idem|

## 6. Conformidade e certificação

|Item|ML-KEM (mlkem-native)|ML-DSA (mldsa-native)|
|---|---|---|
|Conformidade FIPS 203/204|Testada com vetores ACVP e Wycheproof|idem|
|CAVP próprio|Não|Não|
|CMVP (FIPS 140-3)|Não. Nenhum certificado contém este código|Não|

Detalhes sobre o AWS-LC:
- O certificado **#5314** (AWS-LC 3 static, FIPS 140-3 nível 1, validado em 05/06/2026) cobre o módulo **AWS-LC FIPS 3.1.0**. O ML-KEM desse módulo é `ml_kem_ref`, derivado da referência do Kyber (commit `11d00ff`, 26/03/2024), não o mlkem-native. O certificado não lista ML-DSA.
- O certificado **#5429** (AWS-LC dynamic, validado em 20/07/2026) não lista ML-KEM nem ML-DSA.
- O mlkem-native entrou no AWS-LC só na v1.50.0 (abr/2025), depois do ramo FIPS 3.x. Nenhuma tag FIPS do AWS-LC (até a 3.3.0) contém mlkem-native ou mldsa-native.

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

- Maior nível de verificação formal do levantamento: CBMC no C e HOL-Light no assembly de AArch64 e x86_64 (e de RV32IM no mldsa-native).
- Várias arquiteturas: AVX2, NEON, SHA3 no AArch64, RVV, POWER e RV32IM.
- Sem heap, com consumo de memória documentado por operação e modo `REDUCE_RAM` no ML-DSA (sign-87 cai de 107 KB para cerca de 14 KB).
- API completa no ML-DSA: HashML-DSA, external mu, `pk_from_sk` e assinatura reiniciável.
- Integração simples: basta copiar o diretório para o projeto; C90 portátil.
- Licença tripla e permissiva (Apache-2.0, ISC ou MIT).
- Adoção relevante (AWS-LC, libOQS, rustls) e governança pela Linux Foundation.

## 9. Pontos negativos

- Não há validação CMVP, nem indireta (o ML-KEM certificado do AWS-LC é outro código).
- Sem AVX-512 e sem SVE.
- RVV parcial no ML-KEM (NTT só com VLEN = 256) e ausente no ML-DSA.
- Backends RVV, POWER e MVE sem prova formal; testes de tempo constante só em x86_64 e AArch64.
- O tempo constante dos samplers `eta` do ML-DSA ainda não é provado.
- São primitivas apenas: não há TLS, X.509 nem gerência de chaves.
- A v2.0.0 do mlkem-native quebra a API da v1.x.
- Memória alta no ML-DSA na configuração padrão.

## 10. Uso no canal clássico do QKD

- **Autenticação do canal:** o ML-DSA serve para autenticar o canal clássico com certificados ou chaves públicas (modelo "assinatura seguida de MAC"). O ML-KEM entra no modelo "KEM seguido de MAC" ou na troca de chaves híbrida.
- **Embarcados:** sem heap, com memória previsível e em C90 portátil, as duas bibliotecas cabem nos nós de controle do sistema QKD (SoC ARM, RISC-V ou FPGA com processador). Isso permite implementar um protocolo de autenticação próprio diretamente sobre as primitivas.
- **Garantia de correção:** a verificação formal do assembly reduz o risco de erros como o CVE-2026-10097 do wolfSSL, o que pesa num canal cuja segurança depende inteiramente da autenticação.
- **Quando o canal usa TLS:** a forma natural de usar as duas bibliotecas é pelo AWS-LC, que as incorpora. Nesse caso a certificação FIPS do código PQC ainda não está disponível.

## 11. Fontes

**Repositórios e documentação**
- https://github.com/pq-code-package/mlkem-native (README, `mlkem/README.md`, SOUNDNESS.md, RELEASE.md, BIBLIOGRAPHY.md)
- https://github.com/pq-code-package/mldsa-native (README, SOUNDNESS.md, BIBLIOGRAPHY.md)
- Código clonado: mlkem-native `c2827a8`, mldsa-native `a14b01a`
- Anúncio da PQCA: https://www.linuxfoundation.org/press/announcing-the-post-quantum-cryptography-alliance-pqca

**CMVP e AWS-LC**
- Certificado #5314: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/5314
- Security Policy #5314: https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp5314.pdf
- Certificado #5429: https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/5429
- ML-KEM na tag certificada: https://github.com/aws/aws-lc/blob/AWS-LC-FIPS-3.1.0/crypto/fipsmodule/ml_kem/README.md
- Importação atual: https://github.com/aws/aws-lc/blob/main/crypto/fipsmodule/ml_kem/META.yml e https://github.com/aws/aws-lc/blob/main/crypto/fipsmodule/ml_dsa/META.yml
- AWS-LC v1.50.0: https://github.com/aws/aws-lc/releases/tag/v1.50.0

**Verificação formal e canais laterais**
- Palestra "Formally Verified Post-Quantum Cryptography at Scale": https://sos-vo.org/node/109616
- s2n-bignum: https://github.com/awslabs/s2n-bignum
- HOL Light: https://hol-light.github.io/
- CBMC: https://github.com/diffblue/cbmc
- Neon NTT (Auto)formalised: https://eprint.iacr.org/2026/1223
- KyberSlash: https://kyberslash.cr.yp.to/papers.html
- clangover: https://github.com/antoonpurnal/clangover

**Otimizações e memória**
- Neon NTT: https://eprint.iacr.org/2021/986
- SLOTHY: https://eprint.iacr.org/2022/1303
- Keccak híbrido em AArch64: https://eprint.iacr.org/2022/1243
- AVX2 NTT: https://eprint.iacr.org/2018/039
- Dilithium for Memory Constrained Devices: https://eprint.iacr.org/2022/323

---
