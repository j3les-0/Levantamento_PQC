
---
LINKS:
https://botan.randombit.net/
https://github.com/randombit/botan

---
# Botan

Biblioteca criptográfica de uso geral em C++20, com TLS 1.3 e X.509. Tem uma das maiores coberturas de PQC do levantamento (ML-KEM, ML-DSA, SLH-DSA, FrodoKEM, Classic McEliece, XMSS e HSS/LMS) e conta com apoio do BSI (Alemanha).



## 1. Identificação

|Item|Valor|
|---|---|
|Padrões|FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA); também Kyber e Dilithium da 3ª rodada (legado), FrodoKEM, Classic McEliece, XMSS e HSS/LMS|
|Linguagem|C++20|
|Origem do código|Implementação própria, desenvolvida pela Rohde & Schwarz Cybersecurity no projeto do BSI de manutenção e desenvolvimento do Botan (2022 a mar/2025). Código comum Kyber/ML-KEM e Dilithium/ML-DSA (`kyber_common`, `dilithium_common`, `pqcrystals`)|
|Mantenedor / governança|Projeto aberto (Jack Lloyd e colaboradores), com contratos do BSI|
|Versão analisada|3.13.0 (13/08/2026); master `9fc28d5b1` (12/10/2026, 3.14.0 não lançada). Versão revisada pelo BSI: 3.7.1-RSCS1|
|Licença|BSD-2-Clause (Simplified BSD)|
|Integração em protocolos|TLS 1.3 com ML-KEM puro e híbridos (x25519/ML-KEM-768, secp384r1/ML-KEM-1024); X.509; políticas de build `bsi`, `nist` e `fips140`|

## 2. Cobertura funcional

|Característica|ML-KEM|ML-DSA|
|---|---|---|
|Níveis de parâmetro|512 / 768 / 1024|44 / 65 / 87 (modos `ML_DSA_4x4`, `6x5`, `8x7`)|
|Variante pré-hash (HashML-DSA)|Não se aplica|**Não** (o separador de domínio é fixo em 0x00)|
|External mu|Não se aplica|Não|
|Determinística / aleatorizada|Aleatorizada|Hedged por padrão; determinística via `PK_Signature_Options`|
|Formatos de chave|Chave privada guardada como seed (`d‖z`); o formato expandido do FIPS 203 é aceito na importação e preservado|**Só seed (`xi`)**; o formato expandido do FIPS 204 não é aceito|
|Outros recursos|Híbridos para TLS|Contexto validado internamente (≤ 255 bytes); a documentação de `with_context` cita só o SM2|

## 3. Implementação por arquitetura

| Arquitetura            | Extensão                  | ML-KEM | ML-DSA |
| ---------------------- | ------------------------- | ------ | ------ |
| x86_64                 | AVX2                      | ◐     | ○      |
| x86_64                 | AVX-512                   | ◐     | ◐     |
| AArch64                | NEON                      | ○      | ○      |
| AArch64                | SVE/SVE2                  | ○      | ○      |
| AArch64                | SHA3 (Keccak em hardware) | ○      | ○      |
| Arm 32 bits / Cortex-M | Escalar (DSP)             | ○      | ○      |
| Armv8.1-M              | Helium/MVE                | ○      | ○      |
| RISC-V                 | RVV                       | ○      | ○      |
| RISC-V                 | Escalar (RV32/RV64)       | ○      | ○      |
| POWER                  | VSX (ppc64le)             | ○      | ○      |
| s390x                  | Vector / CPACF            | ○      | ○      |

Legenda:

| Marcador | Significado |
| --- | --- |
| ● | As operações principais do algoritmo têm código otimizado para a extensão |
| ◐ | Otimização parcial: só algumas operações, só o Keccak, ou só em versão não lançada ou experimental (detalhes nas notas) |
| ○ | Sem código específico (C portátil) ou arquitetura sem suporte |



Notas por operação:
- **Na 3.13.0 (lançada)**, ML-KEM e ML-DSA são C++ portátil. O ganho existe só na permutação Keccak: versão BMI2 (x86_64 genérico) e versão AVX-512 (`keccak_perm_avx512`), usadas pelos dois algoritmos.
- **x86_64 AVX2 (ML-KEM), só na master/3.14:** aritmética polinomial em AVX2 (`kyber_avx2`, 08/10/2026) e SHAKE com 4 instâncias paralelas (`hash_engine_keccak_avx2`). O ML-DSA não usa essas rotinas.
- **x86_64 AVX-512, só na master/3.14:** SHAKE paralelo em AVX-512 (`hash_engine_keccak_avx512`) para o ML-KEM.
- **Seleção de implementação:** em tempo de execução, por "provider" (`base`, `avx2`, `avx512`).
- **Sem otimização:** não há código para ARM, RISC-V ou POWER.

## 4. Memória e tamanho

|Item|Valor|
|---|---|
|Uso de heap|Sim, intenso (`std::vector`, `secure_vector`, com zeroização e alocador de memória travada opcional)|
|Opções de redução de memória|Nenhuma específica para PQC. Build modular (`configure.py --minimized-build --enable-modules=ml_kem,ml_dsa,...`) reduz o binário|
|Memória por operação documentada|Não|
|Ambientes|SO completos; não é alvo de microcontroladores|

## 5. Segurança

|Item|Valor|
|---|---|
|Verificação formal|Nenhuma|
|Tempo constante|Anotações `CT::poison`/`unpoison` no ML-KEM e no ML-DSA, testadas com valgrind (`src/ct_selftest`)|
|Testes|ACVP (ML-KEM), Wycheproof (verificação ML-DSA), KATs, fuzzers|
|Auditoria|Revisões de código do BSI / Rohde & Schwarz (2017 e 2022–2025), incluindo contramedidas de canal lateral|
|Fora de escopo|Potência/EM e falhas não são declarados|

## 6. Conformidade e certificação

|Item|Valor|
|---|---|
|Conformidade FIPS 203/204|Testada com vetores ACVP (ML-KEM) e Wycheproof (ML-DSA)|
|CAVP|Não|
|CMVP (FIPS 140-3)|Não. O Botan nunca teve módulo validado; o antigo módulo `fips140` foi removido e resta só a política de build `fips140`|
|Avaliação institucional|BSI: revisão da versão 3.7.1-RSCS1 e recomendação da 3.11 para aplicações sensíveis. A 3.7.1-RSCS1 inclui ML-KEM, FrodoKEM, Classic McEliece, ML-DSA, SLH-DSA, XMSS e HSS/LMS, alinhados à BSI TR-02102|

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

- A maior variedade de algoritmos PQC, incluindo FrodoKEM e Classic McEliece, recomendados pelo BSI.
- Código C++ moderno e legível.
- Licença BSD-2 permissiva.
- Revisão independente financiada pelo BSI.
- TLS 1.3 híbrido integrado.
- Build modular.

## 9. Pontos negativos

- Sem HashML-DSA e sem external mu.
- ML-DSA só com chave em seed; contexto do ML-DSA não documentado na API de opções.
- Na versão lançada, nenhuma aritmética otimizada; o AVX2 do ML-KEM existe só na master.
- Nada para ARM, RISC-V ou POWER.
- Uso intenso de heap.
- Sem CMVP, sem CAVP e sem verificação formal.

## 10. Uso no canal clássico do QKD

- **Contexto europeu:** encaixa em infraestrutura alinhada ao BSI (TR-02102), onde a revisão do BSI pode valer mais que a certificação NIST.
- **KEM conservador:** permite combinar ML-KEM com FrodoKEM ou Classic McEliece, o que interessa a quem quer diversidade de suposições matemáticas no canal clássico.
- **Plataforma:** adequado para servidores x86 com TLS 1.3; fraco para nós embarcados ARM ou RISC-V.

## 11. Fontes

- Código clonado: `botan` master `9fc28d5b1` (`src/lib/pubkey/{kyber,dilithium,pqcrystals,sphincsplus}`, `src/lib/permutations/keccak_perm`, `src/lib/hash_engine`, `doc/api_ref/pubkey.rst`, `news.rst`, `src/build-data/policy`, `src/tests/data/pubkey`)
- BSI, Kryptobibliothek Botan: https://www.bsi.bund.de/EN/Themen/Unternehmen-und-Organisationen/Informationen-und-Empfehlungen/Kryptografie/Kryptobibliothek-Botan/kryptobibliothek-botan_node.html
- Documentação da API de chaves públicas: https://botan.randombit.net/handbook/api_ref/pubkey.html
- Notas de versão: https://botan.randombit.net/news.html

---
