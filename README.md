# otp-tool

Command line tool to read and write the OTP (One Time Programmable) memory of Microchip LAN966x and LAN969x SoCs from Linux userspace.

## Supported SoCs

| SoC     | OTP size | Chip ID base | Region non-secure bit |
|---------|----------|--------------|-----------------------|
| LAN966x | 8 KiB    | 0x9660       | No                    |
| LAN969x | 16 KiB   | 0x9690       | Yes                   |

## Build

The tool runs on the target, so it needs a cross compiler. Use one of these:

- The BSP toolchain installed on the host.
- The BSP Docker container, started with the `dr` script.

Option 1, BSP toolchain:

1. Download the toolchain and extract it into `/opt/mchp`, as described in the BSP documentation (Buildroot build, Toolchain).
2. Build with the toolchain file that the toolchain provides in `share/buildroot/toolchainfile.cmake`:

```
cmake -B build -DCMAKE_TOOLCHAIN_FILE=<toolchain-folder>/share/buildroot/toolchainfile.cmake
cmake --build build
```

The toolchain folders are:

- LAN966x (32-bit ARM): `/opt/mchp/<toolchain>/arm-cortex_a8-linux-gnueabihf`
- LAN969x (64-bit ARM): `/opt/mchp/<toolchain>/arm64-armv8_a-linux-gnu`

Option 2, BSP Docker container:

1. Install Docker and the `dr` script from https://github.com/microchip-ung/docker-run.
2. Provide a `.docker.env` file that selects the BSP build image, as in the BSP source tree.
3. Run the build through `dr`:

```
dr sh -c 'mkdir -p build && cd build && cmake .. && make'
```

The binary is `build/otp`. Copy it to the root file system of the target.

Use the `file` command to inspect the build artifact and confirm the target architecture:

```
file build/otp
```

## Requirements

Enable these kernel options:

- `CONFIG_NVMEM_LAN9662_OTPC`: OTP driver for LAN966x and LAN969x.
- `CONFIG_NVMEM_SYSFS`: OTP access through sysfs.

The device tree node needs `reg = <0xe0021000 0x300>` and one of these compatible strings:

- LAN966x: `microchip,lan9662-otpc`, or `"microchip,lan9668-otpc", "microchip,lan9662-otpc"`.
- LAN969x: `microchip,lan969x-otp`.

The tool accesses the OTP through the nvmem file, for example `/sys/bus/nvmem/devices/lan9662-otp0/nvmem`. The `-d` option is required. It selects the nvmem file or a normal file. The file size must equal the OTP size of the SoC (8192 bytes for LAN966x, 16384 bytes for LAN969x).

## Warning

Writes to the OTP cannot be undone. Test every write command first on a copy of the OTP content in a normal file (`-d <file>`), with the same size as the OTP.

## Content types

- Fields: fixed-length data at fixed addresses. The tool has a built-in template per SoC with name, address and length.
- Non-volatile counters: counters that can only be incremented. The secure boot ROM uses them for rollback protection.
- Tagged data: 64-bit records for data such as MAC addresses, board ID and passwords.

Tag record layout:

```
+--------+--------+--------+----------+
| size:3 | cont:1 | tag:12 | value:48 |
+--------+--------+--------+----------+
```

- `size`: number of valid bytes in `value`. The values 0 and 7 are invalid. Writing 7 invalidates the record.
- `cont`: the content continues in the next valid record with the same tag.
- `tag`: tag number, 0-2047.
- `value`: data of the record.

Tag names: `password`, `pcb`, `revision`, `ethaddr`, `ethaddr_count`, `fit_config`, `pcb_2`.

## Regions

The OTP is divided into regions. Each region can be write-protected. On LAN969x each region can also be made accessible to the non-secure world.

| Region | Start  | End    | Content                       |
|--------|--------|--------|-------------------------------|
| 0      | 0x0000 | 0x003f | Manufacturing                 |
| 1      | 0x0040 | 0x0043 | OTP write protect             |
| 2      | 0x0044 | 0x0063 | OTP region definitions        |
| 3      | 0x0064 | 0x00ff | Insecure configuration 1      |
| 4      | 0x0100 | 0x01ff | TBBR keys                     |
| 5      | 0x0200 | 0x023f | Non-volatile secure counters  |
| 6      | 0x0240 | 0x07ff | Insecure configuration 2      |
| 7      | 0x0800 | 0x2000 | User space (tags)             |

## Usage

```
otp field list
otp field get [<NAME>]
otp field set [--merge] <NAME> (ascii|hex|dec|3ascii-dec) <VALUE>
otp addr get <ADDRESS> <BYTE-LENGTH>
otp addr set [--merge] <ADDRESS> <BYTE-LENGTH> (ascii|hex|dec) <VALUE>
otp tag list-tag-names
otp tag print
otp tag info
otp tag get (<tag-number>|<tag-name>)
otp tag del (<tag-number>|<tag-name>)
otp tag set (<tag-number>|<tag-name>) (ascii|hex|dec) <VALUE>
otp nvcnt get (trusted|nontrusted)
otp nvcnt set <VALUE> (trusted|nontrusted)
otp import-keys [--no-randomize-huk] <FILE>
otp region show
otp region write-protect <REGION-NUMBER>
otp region non-secure <REGION-NUMBER>
otp init pcb <PCB> ethaddr (random-ethaddr|<ETHADDR-ADDRESS>) ethaddr_count <COUNT>
```

Options:

| Option                | Description                                                          |
|-----------------------|----------------------------------------------------------------------|
| `-h`, `--help`        | Show the help text.                                                  |
| `--version`           | Show the version.                                                    |
| `--verbose`           | Enable verbose traces.                                               |
| `-d`, `--device DEV`  | Required. nvmem file or normal file of the OTP size.                 |
| `-i`, `--chip-id ID`  | Use the OTP layout of this chip ID.                                  |
| `--chip-id-force`     | Use the `--chip-id` value even if the OTP holds a different part ID. |
| `--no-confirm`        | Do not ask for confirmation before writes.                           |

`--chip-id` is accepted only when the part ID at byte address 0x5-0x6 is not programmed, or when it matches the given ID.

Run `otp --help` for the full command descriptions.

## Write behavior

The commands `field set`, `addr set` and `tag del|set` do the following:

1. Read the existing content.
2. Without `--merge`, check that the value can be written as-is. Abort if not.
3. Print in hex what will be written and where.
4. Ask the user to confirm.

With `--merge`, the new value is OR-ed bitwise with the existing content.

## Examples

Initialize a board (manufacturing only, run once):

```
otp --no-confirm -d /sys/bus/nvmem/devices/lan9662-otp0/nvmem \
    init pcb 8290 ethaddr random-ethaddr ethaddr_count 16
```

This sets the tags `pcb`, `ethaddr` and `ethaddr_count`. A random `ethaddr` has the broadcast bit 0 and the local-administered bit 1.

Read a field:

```
otp -d /sys/bus/nvmem/devices/lan9662-otp0/nvmem field get PARTID
```

The output shows the field length in bytes, the value in little endian hex and the ASCII characters.

Show the regions:

```
otp -d /sys/bus/nvmem/devices/lan9662-otp0/nvmem region show
```

Write-protect region 4 after its data is written (cannot be reverted):

```
otp -d /sys/bus/nvmem/devices/lan9662-otp0/nvmem region write-protect 4
```

Make region 1 accessible to the non-secure world on LAN969x (cannot be reverted):

```
otp -d /sys/bus/nvmem/devices/lan9662-otp0/nvmem region non-secure 1
```

Do not make regions 4 and 5 non-secure.

Use an explicit chip ID when the part ID is not programmed:

```
otp -d /sys/bus/nvmem/devices/lan9662-otp0/nvmem --chip-id 9692 field list
```

## Key import

`otp import-keys <FILE>` programs the TF-A keys into region 4. The input is binary, from a file or from stdin (`-`). The length must match the platform. Unless `--no-randomize-huk` is given, the tool fills `OTP_TBBR_HUK` with random data from `/dev/random`.

## Common fields

| Field            | Size | Description                                                                 |
|------------------|------|-----------------------------------------------------------------------------|
| `SECURE_JTAG`    | 4    | JTAG mode at reset: 0 open, 1 secure mode 1, 2 secure mode 2, 3 closed.     |
| `JTAG_UUID`      | 10   | Unique JTAG identifier of the device.                                       |
| `OTP_SJTAG_SSK`  | 32   | Shared secret for unlocking secure JTAG. Response is `sha256(challenge + OTP_SJTAG_SSK)`. |
| `OTP_TBBR_ROTPK` | 32   | SHA-256 hash of the root of trust public key. Required for secure boot.     |
| `OTP_TBBR_HUK`   | 32   | Hardware unique key. Program with random data.                              |

Run `otp field list` for all fields.

## License

MIT. See `COPYING`.
