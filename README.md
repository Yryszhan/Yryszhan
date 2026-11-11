// ChirpStack v4 — кодек QUASAR Logic v1.1
// Device Profile → Payload codec → JavaScript functions

// Регистры ПЧ NIETZ NZ8000 в порядке CFG_MB_READS (config.h)
var MB_LAYOUT = [
  { names: ["vfd_state"] },                                   // 3000H
  { names: ["freq", "dc_bus_v", "out_v", "out_i",             // 1001H..1007H
            "out_p", "torque", "speed"] },
  { names: ["fault_code"] }                                    // 8000H
];

var MAX_HZ = 50;   // P0.12 — максимальная частота ПЧ

var VFD_STATE = { 1: "run_forward", 2: "run_reverse", 3: "stopped" };

var CMD_RESULT = {
  0x00: "ok", 0x01: "bad_command", 0x02: "register_not_allowed",
  0x03: "modbus_timeout", 0x04: "modbus_error", 0xFF: "none"
};

function u16(b, i) { return (b[i] << 8) | b[i + 1]; }
function i8(v) { return v > 127 ? v - 256 : v; }

function decodeUplink(input) {
  var b = input.bytes, d = {};

  if (input.fPort === 1 && b.length >= 15 && b[0] === 0x01) {
    d.type = "status";
    d.counter = u16(b, 1);
    d.relay1 = (b[3] & 1) !== 0;
    d.relay2 = (b[3] & 2) !== 0;
    d.dout = [];
    d.din = [];
    for (var k = 0; k < 4; k++) {
      d.dout.push((b[4] >> k & 1) === 1);
      d.din.push((b[5] >> k & 1) === 1);
    }
    d.ain_mv = [u16(b, 6), u16(b, 8), u16(b, 10), u16(b, 12)];
    d.last_cmd = (b[14] & 0x80) && b[14] !== 0xFF
      ? "modbus_exception_" + (b[14] & 0x7F)
      : (CMD_RESULT[b[14]] || ("code_" + b[14]));
    return { data: d };
  }

  if (input.fPort === 2 && b.length >= 3 && b[0] === 0x02) {
    d.type = "modbus";
    d.ok_mask = b[1];
    d.error = i8(b[2]);              // 0 ok, -1 таймаут, -2 CRC, -3 кадр, >0 исключение
    var pos = 3;
    for (var q = 0; q < MB_LAYOUT.length; q++) {
      var ok = (b[1] >> q & 1) === 1;
      for (var r = 0; r < MB_LAYOUT[q].names.length; r++) {
        if (pos + 1 >= b.length) break;
        if (ok) d[MB_LAYOUT[q].names[r]] = u16(b, pos);
        pos += 2;
      }
    }
    if (d.vfd_state !== undefined) d.vfd_state_text = VFD_STATE[d.vfd_state] || "unknown";
    return { data: d };
  }

  return { data: { raw: b } };
}

// Команды (FPort 10). Примеры JSON для Queue → "JSON":
//   {"relays": 3}                     -> 01 03
//   {"status": true}                  -> 02
//   {"dout": 5}                       -> 03 05
//   {"vfd": "start"} / "stop" / "reset"  -> 04 20 00 00 01 / 00 06 / 00 07
//   {"freq_hz": 25}                   -> 04 10 00 13 88  (25/50 = 50.00%)
//   {"freq_raw": 5000}                -> 04 10 00 13 88  (сотые доли %)
//   {"modbus_write": {"reg": 4096, "value": 1}}
//   {"poll": true}                    -> 05
//   {"interval": {"status": 300, "modbus": 60}}  -> 06 01 2C 00 3C  (0 = не менять)
function encodeDownlink(input) {
  var d = input.data;
  if (d.relays !== undefined) return { fPort: 10, bytes: [0x01, d.relays & 3] };
  if (d.status) return { fPort: 10, bytes: [0x02] };
  if (d.dout !== undefined) return { fPort: 10, bytes: [0x03, d.dout & 15] };
  if (d.vfd !== undefined) {
    var c = { start: 1, reverse: 2, coast: 5, stop: 6, reset: 7 }[d.vfd];
    if (c === undefined) return { errors: ["unknown vfd command"] };
    return { fPort: 10, bytes: [0x04, 0x20, 0x00, 0x00, c] };
  }
  if (d.freq_hz !== undefined) d.freq_raw = Math.round(d.freq_hz / MAX_HZ * 10000);
  if (d.freq_raw !== undefined) {
    if (d.freq_raw < 0 || d.freq_raw > 10000) return { errors: ["freq out of range"] };
    return { fPort: 10, bytes: [0x04, 0x10, 0x00, (d.freq_raw >> 8) & 255, d.freq_raw & 255] };
  }
  if (d.modbus_write) {
    var w = d.modbus_write;
    return { fPort: 10, bytes: [0x04, (w.reg >> 8) & 255, w.reg & 255, (w.value >> 8) & 255, w.value & 255] };
  }
  if (d.poll) return { fPort: 10, bytes: [0x05] };
  if (d.interval) {
    var st = d.interval.status || 0, mb = d.interval.modbus || 0;
    if ((st && st < 10) || (mb && mb < 10) || st > 65535 || mb > 65535) return { errors: ["interval 10..65535 s"] };
    return { fPort: 10, bytes: [0x06, (st >> 8) & 255, st & 255, (mb >> 8) & 255, mb & 255] };
  }
  return { errors: ["unknown command"] };
}

<h1 align="center">Yryszhan Sansyzbay</h1>
<p align="center">
Embedded Systems Engineer — STM32 · ESP32 · Raspberry Pi · LoRaWAN · Computer Vision · PCB Design
</p>

---

### About

- Computer Engineering student, Yessenov University, Aktau, Kazakhstan
- Focus areas: embedded systems, IoT, computer vision, PCB design
- Current project: LoRaWAN Industrial Node Controller (STM32, KiCad, ChirpStack)
- Interests: STM32 HAL/LL/BARE METAL, FreeRTOS, KiCad, ROS2, YOLOv8, TFLite, Buildroot

---

### Contact

- LinkedIn: [yryszhan-sansyzbay](https://www.linkedin.com/in/yryszhan-sansyzbay-6a9b26270/)
- Instagram: [@yryszhan.engineer](https://www.instagram.com/yrys_szh/)
- Email: yryszhan232@gmail.com

---

### Stack

<p>
  <img src="https://img.shields.io/badge/C/C++-00599C?logo=c&logoColor=white" />
  <img src="https://img.shields.io/badge/STM32-03234B?logo=stmicroelectronics&logoColor=white" />
  <img src="https://img.shields.io/badge/ESP32-000000?logo=espressif&logoColor=white" />
  <img src="https://img.shields.io/badge/Raspberry%20Pi-A22846?logo=raspberrypi&logoColor=white" />
  <img src="https://img.shields.io/badge/ROS2-22314E?logo=ros&logoColor=white" />
  <img src="https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/PyTorch-EE4C2C?logo=pytorch&logoColor=white" />
  <img src="https://img.shields.io/badge/OpenCV-5C3EE8?logo=opencv&logoColor=white" />
  <img src="https://img.shields.io/badge/KiCad-314CB0?logo=kicad&logoColor=white" />
  <img src="https://img.shields.io/badge/Fusion%20360-FC6D26?logo=autodesk&logoColor=white" />
  <img src="https://img.shields.io/badge/FreeRTOS-2C3E50?logo=freertos&logoColor=white" />
</p>

---

### Recognition

Elbasy Award recipient (bronze, silver). WRO finalist. Fab Academy 2025 graduate (MIT CBA network). Regional engineering competition laureate.
