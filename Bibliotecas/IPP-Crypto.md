
---
LINKS:
https://github.com/intel/cryptography-primitives
https://www.intel.com/content/www/us/en/developer/tools/oneapi/ipp-crypto.html
https://pypi.org/project/ipp-crypto/

---
# IPP-Crypto (Intel® Cryptography Primitives Library)

Biblioteca de primitivas criptográficas da Intel, antes "Intel IPP Cryptography", também distribuída no oneAPI (2026.0). É uma coleção de *building blocks* com despacho por geração de CPU, não uma biblioteca de protocolo. Tem ML-KEM, ML-DSA, LMS e XMSS.


## 1. Identificação

|Item|Valor|
|---|---|
|Padrões|FIPS 203 (ML-KEM), FIPS 204 (ML-DSA); LMS (assinatura e verificação) e XMSS|
|Linguagem|C, com caminhos de código por geração de CPU: `y8` SSE4.2, `l9` AVX2, `k0`/`k1` AVX-512, `d1` AVX10.2|
|Origem do código|Implementação própria da Intel: ML-KEM na 1.3.0, otimizado com AVX-512 na 1.4.0; ML-DSA na 1.4.0 / oneAPI 2026.0|
|Mantenedor / governança|Intel (código aberto no GitHub; branches `release` e `develop`)|
|Versão analisada|v2.3.0 (16/07/2026); develop `aabec7cbb` (05/10/2026, com a 2.4.0 no CHANGELOG); pacote oneAPI 2026.0 (29/04/2026)|
|Licença|Apache-2.0|
|Integração em protocolos|Nenhuma (só primitivas)|

**Status da API:** experimental. As funções de ML-KEM e ML-DSA só ficam disponíveis definindo `IPPCP_PREVIEW_ML_KEM` / `IPPCP_PREVIEW_ML_DSA` antes de incluir `ippcp.h`; a documentação diz "The API family is supported in experimental mode".

## 2. Cobertura funcional

|Característica|ML-KEM|ML-DSA|
|---|---|---|
|Níveis de parâmetro|512 / 768 / 1024 (`IPPCP_ML_KEM_*`)|44 / 65 / 87 (`ML_DSA_*`)|
|Variante pré-hash (HashML-DSA)|Não se aplica|**Não**|
|External mu|Não se aplica|**Só na verificação** (`ippsMLDSA_Verify_Mu`, desde a 2.3.0)|
|Determinística / aleatorizada|Aleatorizada; RNG interno com RDSEED/RDRAND (2.3.0) ou fornecido pelo usuário (`rndFunc`)|A assinatura recebe um `rndFunc` do usuário; não há modo determinístico explícito; contexto suportado|
|Formatos de chave|Formato FIPS 203|Formato FIPS 204|
|Outros recursos|Consulta de tamanho de buffers por operação|`maxMessageLength` definido no `Init`; `IPPCP_MLDSA_NO_MESSAGE` para uso só com `Verify_Mu`|

## 3. Implementação por arquitetura


| Arquitetura            | Extensão                  | ML-KEM | ML-DSA |
| ---------------------- | ------------------------- | ------ | ------ |
| x86_64                 | AVX2                      | ●     | ◐     |
| x86_64                 | AVX-512                   | ●     | ◐     |
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
- **Só x86 (Intel 64, Linux e Windows):** a biblioteca é otimizada para CPUs Intel e não tem suporte a ARM, RISC-V, POWER ou s390x.
- **AVX2 (`l9`):**
  - ML-KEM: operações polinomiais e geração da matriz A.
  - ML-DSA: só a amostragem (por exemplo, `RejNTTPoly`); a NTT fica em C.
  - O SHAKE com 4 instâncias paralelas em AVX2, compartilhado pelos dois algoritmos, existe só no branch `develop` (2.4.0).
- **AVX-512 (`k0`/`k1`):**
  - ML-KEM: keygen, encaps e decaps otimizados (1.4.0).
  - ML-DSA: só o Keccak, que tem implementação AVX-512 no módulo SHA-3.
- **AVX10.2 (`d1`):** há caminho de despacho desde a 2.0.0, sem código PQC específico identificado.

## 4. Memória e tamanho

|Item|Valor|
|---|---|
|Uso de heap|Não interno. O usuário consulta o tamanho do contexto e dos buffers (`ippsMLKEM_GetSize`, `*_BufferGetSize`, `ippsMLDSA_*BufferGetSize`) e fornece a memória|
|Opções de redução de memória|Modo `CP_ML_KEM_MEMORY_OPTIMIZED` no ML-KEM|
|Memória por operação documentada|Sim, por função de consulta em tempo de execução e pelas fórmulas de `memory_consumption.h`|
|Ambientes|Linux (RHEL, SLES, Ubuntu, Fedora, Debian, Rocky, Amazon Linux) e Windows, em CPUs Intel Core, Core Ultra, Xeon e Atom|

## 5. Segurança

|Item|Valor|
|---|---|
|Verificação formal|Nenhuma|
|Tempo constante|Testes com o *Pin-based Constant Execution Checker* nos branches `release`, mas ML-KEM e ML-DSA não estão na lista de funções testadas (`CONST_TIME_EXECUTION_TESTING.md`)|
|Testes|Autotestes FIPS de ML-DSA (desde a 2.1.0) e de ML-KEM|
|Correções recentes|2.3.0: compressão do texto cifrado do ML-KEM tornada de tempo constante; zeroização de segredos na pilha em caminhos de erro; correção de escrita fora dos limites na amostragem por rejeição do ML-DSA. 2.2.0: validação estrita do peso do vetor de dicas no ML-DSA; correção de leitura fora dos limites no SHA3 AVX-512 incremental|
|Fora de escopo|Potência/EM e falhas|

## 6. Conformidade e certificação

|Item|Valor|
|---|---|
|Conformidade FIPS 203/204|Implementação segundo os FIPS finais, com autotestes FIPS|
|CAVP|Certificados citados no README para as versões AVX-512 da ippcp e da crypto_mb; não foi identificado CAVP de ML-KEM/ML-DSA|
|CMVP (FIPS 140-3)|Não. O `README_FIPS.md` define a biblioteca como "FIPS-Compliant", não "FIPS-Certified"; a certificação fica a cargo da aplicação que a usa|
|Indicador FIPS|`ippcp_is_fips_approved_func`|

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

- ML-KEM com AVX2 e AVX-512 específico para CPUs Intel.
- Sem heap interno e com memória previsível (buffers fornecidos pelo usuário, tamanhos consultáveis).
- Autotestes FIPS e indicador de serviço aprovado.
- LMS (assinatura e verificação) e XMSS na mesma biblioteca.
- Licença Apache-2.0.

## 9. Pontos negativos

- API de ML-KEM e ML-DSA ainda experimental (preview).
- Só x86.
- Sem HashML-DSA; external mu só na verificação.
- ML-DSA com otimização apenas parcial.
- ML-KEM e ML-DSA fora do escopo dos testes de tempo constante.
- Sem verificação formal e sem CMVP.
- Sem integração de protocolo.

## 10. Uso no canal clássico do QKD

- **Cenário possível:** servidores Intel Xeon em que o desempenho do ML-KEM seja crítico (por exemplo, um nó concentrador com muitos enlaces), com a biblioteca usada como primitiva por baixo de outra pilha de protocolo.
- **Limitação principal:** o status experimental da API e a ausência de testes de tempo constante para PQC desaconselham o uso em produção no canal de autenticação de um sistema QKD hoje.

## 11. Fontes

- Código clonado: `cryptography-primitives` develop `aabec7cbb` (`sources/ippcp/ml_kem`, `sources/ippcp/ml_dsa`, `sources/include/stateless_pqc`, `include/ippcp.h`, `include/ippcpdefs.h`, `CHANGELOG.md`, `README_FIPS.md`, `CONST_TIME_EXECUTION_TESTING.md`, `doc/source/ml_kem`, `doc/source/ml_dsa`)
- Notas de versão 2026: https://www.intel.com/content/www/us/en/developer/articles/release-notes/cryptography/2026.html
- Guia do desenvolvedor (ML-KEM): https://www.intel.com/content/www/us/en/docs/crypto-primitives-library/developer-guide-reference/2026-0/index-2.html
- Pin-based Constant Execution Checker: https://github.com/intel/pin_based_cec

---
