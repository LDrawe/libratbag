# Guia de Testes no Linux — Pichau P301 (`libratbag`)

> [!NOTE]
> Este arquivo e a pasta `p301-notes/` foram adicionados em um commit separado (`docs: add Pichau P301 backup and Linux test guide (drop before PR)`) apenas para você ter o backup e as instruções disponíveis ao trocar para o Linux.
> **Antes de abrir o Pull Request para `libratbag/libratbag`**, remova esse commit temporário da branch (veja a seção final).

---

## 1. O que já foi implementado na branch `sinowealth-pichau-p301`

1. **Sensor `A3050` e tabela discreta de DPI em `src/driver-sinowealth.c`:**
   - Adicionado `SINOWEALTH_SENSOR_A3050 = 0x02` em `enum sinowealth_sensor`.
   - Adicionada a tabela discreta `SINOWEALTH_A3050_DPIS` (`raw = 0x01..0x0b`):
     `{ 500, 750, 1000, 1250, 1500, 1750, 2000, 2500, 3000, 3500, 4000 }`.
   - Atualizadas `sinowealth_get_max_dpi_for_sensor` (máximo `4000`), `sinowealth_raw_to_dpi` (com clamp para slots desativados) e `sinowealth_dpi_to_raw` (mapeamento para o índice `1..11`).
   - Atualizada `sinowealth_init_profile` para expor apenas os 11 valores discretos do `A3050` em `ratbag_resolution_set_dpi_list` e não habilitar `RATBAG_RESOLUTION_CAP_SEPARATE_XY_RESOLUTION` nesse sensor.

2. **Entrada do Pichau P301 (`X182`) em `data/devices/sinowealth-1007.device`:**
   ```ini
   # Configuration size: 123 bytes.
   [Driver/sinowealth/devices/X182]
   Buttons=8
   DeviceName=Pichau P301
   LedType=RGB
   Profiles=2
   SensorType=A3050
   ```

3. **Verificações já executadas no Windows:**
   - `git diff --check` passou sem erros de formatação/espaços em branco.
   - `python test/data-parse-test.py data/devices` passou para todos os arquivos `.device`.
   - A compilação em C (`meson`/`ninja`) e os testes com o mouse real ainda não foram rodados e precisam ser feitos no Linux.

---

## 2. Como puxar, compilar e testar com segurança no Linux

### Passo 1 — Clonar ou atualizar na branch `sinowealth-pichau-p301`
```bash
git clone -b sinowealth-pichau-p301 https://github.com/LDrawe/libratbag.git
cd libratbag
```
*(Ou, se já tiver o repo no Linux: `git fetch origin && git checkout -b sinowealth-pichau-p301 origin/sinowealth-pichau-p301`)*

### Passo 2 — Compilar e rodar os testes unitários
*(Certifique-se de ter as dependências de build do `libratbag` instaladas: `meson`, `ninja`, `pkg-config`, `libudev`, `libevdev`, `glib-2.0`/`systemd`, `json-glib`, `libunistring`, `swig`/`python3` conforme sua distro).*
```bash
meson setup build
ninja -C build
meson test -C build
```

### Passo 3 — Subir o `ratbagd` local com log completo (sem sobrescrever o instalado no sistema)
Pare temporariamente o serviço do sistema e execute o binário compilado em modo verbose:
```bash
sudo systemctl stop ratbagd
sudo ./build/ratbagd --verbose=raw
```

### Passo 4 — Testar primeiro SÓ LEITURA (em outro terminal)
```bash
./build/ratbagctl list
./build/ratbagctl "Pichau P301" info
```
Verifique se o `info` mostra:
- Nome `Pichau P301`
- 2 perfis (`Profile 0` e `Profile 1`)
- 8 botões
- DPIs discretos (`500, 750, 1000, 1250, 1500, 1750, 2000, 2500, 3000, 3500, 4000`)

Se a leitura estiver 100% correta, você pode abrir o `piper` enquanto esse `./build/ratbagd` estiver rodando.

### Passo 5 — Pontos específicos para observar no log `--verbose=raw`
1. **Tamanho da leitura (`GET_FEATURE`):** confira se `sinowealth_read_raw_configs` aceitou o tamanho devolvido pelo `hidraw` (precisa estar entre `123` e `167` bytes).
2. **Escrita (`config_write`):** hoje o `driver-sinowealth.c` envia `0x01` no byte `0x03`, enquanto o `OemDrv.exe` escrevia `0x7b` no byte `0x06` e `0x00` no byte `0x03`.
   - Ao testar **uma primeira alteração simples** (por ex., mudar uma cor ou DPI no **Perfil 2 / Profile 1 no `ratbagctl`**, deixando o **Perfil 1 / Profile 0** intacto por precaução), rode `./build/ratbagctl "Pichau P301" info` depois para conferir se a mudança foi gravada e mantida.
   - Se o mouse ignorar a escrita ou der erro no log, verifique esse offset (`0x03` vs `0x06`) em `src/driver-sinowealth.c`.
   - Se algo corromper o Perfil 1, use o backup e o script em `p301-notes/p301_backup_e_protocolo.md` para restaurar.

---

## 3. Como limpar os arquivos temporários antes de abrir o Pull Request

Depois que os testes no Linux estiverem 100% validados e você tiver terminado eventuais ajustes de código:

1. Se você fez commits novos depois dos testes, garanta que `p301-notes/` seja removido da branch antes do PR:
   - Se **não** fez novos commits em cima do commit `docs: ...`:
     ```bash
     git reset --hard HEAD~1
     git push --force-with-lease origin sinowealth-pichau-p301
     ```
   - Ou, se **fez** outros commits depois (mais simples e sem rebase):
     ```bash
     git rm -r p301-notes
     git commit --amend # ou faça squash dos commits locais antes do PR
     ```
2. Para acompanhar o repositório oficial (`upstream`):
   ```bash
   git remote add upstream https://github.com/libratbag/libratbag.git
   git fetch upstream
   ```
3. Depois do `push` limpo (contendo apenas `src/driver-sinowealth.c` e `data/devices/sinowealth-1007.device`), abra o Pull Request de `LDrawe/libratbag:sinowealth-pichau-p301` para `libratbag/libratbag:master`.
