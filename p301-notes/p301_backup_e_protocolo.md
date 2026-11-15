# Pichau P301 — Backup Completo & Protocolo USB HID (`X182`)

> [!IMPORTANT]
> Este documento contém o mapa do protocolo USB HID extraído do `OemDrv.exe` / `Cfg.ini` e o **dump limpo funcional** da memória onboard (EEPROM) dos **Perfis 1 e 2** do seu Pichau P301, além de um script de recuperação de emergência caso algum teste no `libratbag` corrompa o mouse novamente.

---

## 1. Identificação do Hardware e Firmware

| Campo | Valor Observado |
| :--- | :--- |
| **Modelo** | Pichau Gaming P301 (`PGM-P301-RGB`) |
| **USB VID / PID / REV** | `0x258A` / `0x1007` / `0x0101` |
| **USB Manufacturer / Product** | `'SINOWEALTH'` / `'Wired Gaming Mouse'` |
| **Assinatura Firmware (`Cmd 0x01` / `PSD`)** | `58 31 38 32` (`"X182"`) |
| **Sensor Físico** | Avago 3050 (`0x3050`, código `0x02` na tabela `GetSenosr`) |
| **Perfis Onboard** | 2 perfis (`0x11` = Perfil 1, `0x21` = Perfil 2; bloco `0x31` não existe neste firmware) |

> [!WARNING]
> No `Cfg.ini` oficial, o arquivo lista `PID=0x0028` e `AutoSw=0`, mas o hardware real usa `PID=0x1007`. O `OemDrv.exe` lê sempre o **Perfil 1 (`0x11`)** ao iniciar: se o byte de sensor (`raw[0x09]`) for alterado de `0x02` (por ex., para `0x01` = `0x5050`), o app trava com `Sensor unmatch: dev=0x5050, cfg=0x3050`.

---

## 2. Interfaces HID (`VID_258A&PID_1007&MI_01`)

| Coleção | UsagePage / Usage | Tamanhos | Função |
| :--- | :--- | :--- | :--- |
| `col03` | `0xFF00` / `0x0001` | `Feat = 520` | Leitura e escrita dos blocos de perfil (`Report ID = 0x04`) |
| `col05` | `0xFF00` / `0x0001` | `Feat = 6` | Comandos de controle/consulta (`Report ID = 0x05`) |
| `col04` | `0xFF00` / `0x0001` | `In = 8` | Notificações / eventos vendor |

---

## 3. Protocolo Extraído do `OemDrv.exe`

### 3.1 Comandos (`Report ID = 0x05`, 6 bytes via `HidD_SetFeature` / `HidD_GetFeature` em `col05`)

* **Consultar assinatura / firmware (`0x01`):**
  * Envia: `05 01 00 00 00 00`
  * Lê: `05 01 58 31 38 32` (`"X182"`)
* **Consultar perfil ativo (`0x02`):**
  * Envia: `05 02 00 00 00 00`
  * Lê: `05 02 01 00 00 00` (Perfil 1 ativo) ou `05 02 02 00 00 00` (Perfil 2 ativo)
* **Trocar perfil ativo (`0x02`):**
  * Envia: `05 02 <perfil> 00 00 00` (`<perfil>` = `0x01` ou `0x02`)
* **Selecionar bloco para leitura em `col03` (`0x11` / `0x21`):**
  * Envia: `05 11 00 00 00 00` (Perfil 1) ou `05 21 00 00 00 00` (Perfil 2)
  * Em seguida lê `520` bytes via `HidD_GetFeature(ReportID=0x04)` em `col03`.

### 3.2 Estrutura do Bloco de Perfil (`Report ID = 0x04`, 520 bytes)

* **Cabeçalho na leitura (`raw[0x00..0x07]`):** `04 <cmd: 11|21> 00 00 00 00 06 00`
* **Cabeçalho na escrita (`WriteBlock` em `0x4131c0`):** `04 <cmd: 11|21> 00 00 00 00 7b 00` (payload útil de `0x7b` = **123 bytes** em `raw[0x08..0x82]`, restante preenchido com `0x00` até 520 bytes).

| Offset (`raw`) | Offset (`out`) | Significado Observado |
| :--- | :--- | :--- |
| `0x08` | `0x00` | Constante `0x64` (100) |
| **`0x09`** | **`0x01`** | **ID do sensor** (`0x01`=`0x5050`, **`0x02`=`0x3050`**, `0x03`=`0x9800`, `0x04`=`0x3320`, `0x05`=`0x3330`, `0x06`=`0x3360`, `0x07`=`0x3610`, `0x08`=`0x3212`, `0x09`=`0x3310`, `0x0a`=`0x3325`, `0x0b`=`0x7350`, `0x0c`=`0x3327`). **Nunca sobrescrever `0x02`!** |
| `0x0a` | `0x02` | Polling rate + flags (`0x01`=125Hz, `0x02`=500Hz, `0x03`=250Hz, `0x04`=1000Hz; bit 7 `0x80` usado por opção no `OemDrv.exe`) |
| `0x0b` | `0x03` | Nibble alto = estágio de DPI ativo (`1..6`), nibble baixo = quantidade de estágios `DM` (`6`) |
| `0x0c` | `0x04` | Máscara de estágios desabilitados (`0xc0`) |
| `0x0d..0x12` | `0x05..0x0a` | Índices dos 6 estágios de DPI (padrão observado: `01 03 05 07 09 0b`) |
| `0x1d..0x2e` | `0x15..0x26` | Cores RGB dos 6 estágios de DPI (`6 × 3` bytes: `ff00ff`, `ff0000`, `00ffff`, `00ff00`, `0000ff`, `ffff00`) |
| `0x2f..0x82` | `0x27..0x7a` | Parâmetros de LED (modo/brilho/velocidade/cor) e mapa de botões (`0x7b` bytes totais de payload até `0x82`) |

### 3.3 Tabela discreta de DPI do A3050 (`0x3050`)

Valores `raw` `0x01..0x0b` mapeiam em ordem 1-indexed para:
`500, 750, 1000, 1250, 1500, 1750, 2000, 2500, 3000, 3500, 4000`

---

## 4. Dumps Limpos Salvos (123 bytes de payload `raw[0x08..0x82]`)

### Perfil 1 limpo (`0x11` — estado atual funcionando após ajuste)
```text
64 02 04 46 c0 01 03 05 07 09 0b 00 00 00 00 00
00 00 00 00 00 ff 00 ff ff 00 00 00 ff ff 00 ff
00 00 00 ff ff ff 00 00 00 00 00 00 00 01 42 01
30 ff 00 00 32 07 ff 00 00 00 00 ff 00 ff 00 80
00 ff fa c6 03 fa 03 6a fa 00 fa 02 32 00 ff 00
00 00 ff 00 00 00 ff ff ff 00 00 ff ff ff ff ff
fa 00 ff 00 00 00 32 ff 00 00 00 00 ff 32 32 00
00 00 00 00 00 00 00 00 00 00 00
```

### Perfil 2 limpo (`0x21` — padrão de fábrica intacto)
```text
64 02 02 16 c0 01 03 05 07 09 0b 00 00 00 00 00
00 00 00 00 00 ff 00 ff ff 00 00 00 ff ff 00 ff
00 00 00 ff ff ff 00 ff ff ff 00 00 00 01 02 00
40 ff 00 ff 02 07 ff 00 00 00 ff 00 00 00 ff 00
ff ff ff ff 00 ff 00 ff ff ff ff 02 02 00 ff 00
00 00 ff 00 00 00 ff ff ff 00 00 ff ff ff ff ff
fa 00 ff 00 00 00 02 00 ff 00 fa 03 6a 02 02 00
00 00 00 00 00 00 00 00 00 00 00
```

---

## 5. Script de Recuperação de Emergência

### 5.1 Windows (Python puro, sem dependências externas)
Salve como `restore_p301.py` e execute `python restore_p301.py` se o mouse voltar a travar o sensor:

```python
import ctypes
import time
from ctypes import wintypes

P1_CLEAN = bytes.fromhex(
    "64 02 04 46 c0 01 03 05 07 09 0b 00 00 00 00 00 "
    "00 00 00 00 00 ff 00 ff ff 00 00 00 ff ff 00 ff "
    "00 00 00 ff ff ff 00 00 00 00 00 00 00 01 42 01 "
    "30 ff 00 00 32 07 ff 00 00 00 00 ff 00 ff 00 80 "
    "00 ff fa c6 03 fa 03 6a fa 00 fa 02 32 00 ff 00 "
    "00 00 ff 00 00 00 ff ff ff 00 00 ff ff ff ff ff "
    "fa 00 ff 00 00 00 32 ff 00 00 00 00 ff 32 32 00 "
    "00 00 00 00 00 00 00 00 00 00 00"
)

P2_FACTORY = bytes.fromhex(
    "64 02 02 16 c0 01 03 05 07 09 0b 00 00 00 00 00 "
    "00 00 00 00 00 ff 00 ff ff 00 00 00 ff ff 00 ff "
    "00 00 00 ff ff ff 00 ff ff ff 00 00 00 01 02 00 "
    "40 ff 00 ff 02 07 ff 00 00 00 ff 00 00 00 ff 00 "
    "ff ff ff ff 00 ff 00 ff ff ff ff 02 02 00 ff 00 "
    "00 00 ff 00 00 00 ff ff ff 00 00 ff ff ff ff ff "
    "fa 00 ff 00 00 00 02 00 ff 00 fa 03 6a 02 02 00 "
    "00 00 00 00 00 00 00 00 00 00 00"
)

setupapi = ctypes.windll.setupapi
hid = ctypes.windll.hid
kernel32 = ctypes.windll.kernel32

class GUID(ctypes.Structure):
    _fields_ = [
        ("Data1", wintypes.DWORD),
        ("Data2", wintypes.WORD),
        ("Data3", wintypes.WORD),
        ("Data4", ctypes.c_ubyte * 8),
    ]

class SP_DEVICE_INTERFACE_DATA(ctypes.Structure):
    _fields_ = [
        ("cbSize", wintypes.DWORD),
        ("InterfaceClassGuid", GUID),
        ("Flags", wintypes.DWORD),
        ("Reserved", ctypes.c_size_t),
    ]

setupapi.SetupDiGetClassDevsW.restype = ctypes.c_void_p
setupapi.SetupDiEnumDeviceInterfaces.argtypes = [ctypes.c_void_p, ctypes.c_void_p, ctypes.POINTER(GUID), wintypes.DWORD, ctypes.POINTER(SP_DEVICE_INTERFACE_DATA)]
setupapi.SetupDiGetDeviceInterfaceDetailW.argtypes = [ctypes.c_void_p, ctypes.POINTER(SP_DEVICE_INTERFACE_DATA), ctypes.c_void_p, wintypes.DWORD, ctypes.POINTER(wintypes.DWORD), ctypes.c_void_p]
setupapi.SetupDiDestroyDeviceInfoList.argtypes = [ctypes.c_void_p]
kernel32.CreateFileW.restype = ctypes.c_void_p
hid.HidD_SetFeature.argtypes = [ctypes.c_void_p, ctypes.c_void_p, wintypes.ULONG]
kernel32.CloseHandle.argtypes = [ctypes.c_void_p]

guid = GUID()
hid.HidD_GetHidGuid(ctypes.byref(guid))
dev_info = setupapi.SetupDiGetClassDevsW(ctypes.byref(guid), None, None, 0x12)
iface_data = SP_DEVICE_INTERFACE_DATA()
iface_data.cbSize = ctypes.sizeof(SP_DEVICE_INTERFACE_DATA)
idx = 0
col3 = col5 = None
while setupapi.SetupDiEnumDeviceInterfaces(dev_info, None, ctypes.byref(guid), idx, ctypes.byref(iface_data)):
    idx += 1
    req = wintypes.DWORD(0)
    setupapi.SetupDiGetDeviceInterfaceDetailW(dev_info, ctypes.byref(iface_data), None, 0, ctypes.byref(req), None)
    buf = ctypes.create_string_buffer(req.value)
    ctypes.cast(buf, ctypes.POINTER(wintypes.DWORD))[0] = 8 if ctypes.sizeof(ctypes.c_void_p) == 8 else 6
    if setupapi.SetupDiGetDeviceInterfaceDetailW(dev_info, ctypes.byref(iface_data), buf, req.value, None, None):
        path = ctypes.wstring_at(ctypes.addressof(buf) + 4)
        if "vid_258a&pid_1007" in path.lower():
            if "col03" in path.lower(): col3 = path
            if "col05" in path.lower(): col5 = path
setupapi.SetupDiDestroyDeviceInfoList(dev_info)

assert col3 and col5, "Pichau P301 (258A:1007) nao encontrado!"
h3 = kernel32.CreateFileW(col3, 0xC0000000, 3, None, 3, 0, None)
h5 = kernel32.CreateFileW(col5, 0xC0000000, 3, None, 3, 0, None)

def write_block(cmd, payload):
    pkt = bytearray(520)
    pkt[0] = 0x04
    pkt[1] = cmd
    pkt[6] = 0x7b
    pkt[8:8 + len(payload)] = payload
    ok = hid.HidD_SetFeature(h3, ctypes.create_string_buffer(bytes(pkt), 520), 520)
    time.sleep(0.2)
    return ok

print("Restaurando Perfil 1 (0x11):", write_block(0x11, P1_CLEAN))
print("Restaurando Perfil 2 (0x21):", write_block(0x21, P2_FACTORY))
ok_p1 = hid.HidD_SetFeature(h5, ctypes.create_string_buffer(bytes([0x05, 0x02, 0x01, 0, 0, 0]), 6), 6)
print("Ativando Perfil 1:", ok_p1)

kernel32.CloseHandle(h3)
kernel32.CloseHandle(h5)
```

### 5.2 Linux (`/dev/hidraw*`)
No Linux, os mesmos pacotes (`520` bytes com `Report ID = 0x04` e `6` bytes com `Report ID = 0x05`) podem ser enviados via `ioctl` `HIDIOCSFEATURE` no nó `/dev/hidraw*` correspondente à interface `MI_01` do `258a:1007` (o `hidraw` cujo report descriptor tenha Feature Report `0x04` de 520 bytes e `0x05` de 6 bytes).
