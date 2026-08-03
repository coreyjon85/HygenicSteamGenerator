# Steam Generator Project
DIY Low cost low pressure culinary steam generator for steaming rice.

# Disclaimer: 
None of this is necessary. Some contactors, a few switches, some old school mechanical back-ups (over pressure/vacuum relief), maybe a few other odds and ends - but that would probably get you close. This is a dive down the rabbit hole to explore safety, control, Data in the brewery. This is excessive and no one should follow this. Also, the end goal is to make steam, and even low pressure steam can do some serious bodily harm. Be safe, be smart. 

# PLC control, safety monitoring, logging, and data display.
+ Using Beckoff and Ifm equipment, bring in hygienic sensor data over IO Link to EtherCAT to PLC interface. Run local control/safety loop. Transmit/Log/Display relevant information through MQTT/JSON to Home Assistant to log data in Influx DB for display in Grafana. 

# The Plan
+ Beckoff C6015: running Ubuntu Pro +PREEMPT_RT
+ IgH Ethercat Master: Controlling the fieldbus
+ Beckoff EtherCAT I/O (EL Terminals)
+ Structured IEC 61131-3 Control Logic
+ MQTT/ADS Bridge to publish process data
+ InfluxDB + Grafana: running on separate server to display dashboards
+ Home Assistant: Non-Critical monitoring & notifications

# PLC Setup
+ Beckoff C6015-0010 Industrial PC.
  + Installed Ubuntu Server 22.04 (ensure the version supports realtime-kernel).
  + Ubuntu Pro needs 22.04 LTS  Jammy Jellyfish for real time kernel support.
  + Had to disable network cloud config, and manually create a correct netplan to get ethernet working
  + Sudo apt update && upgrade
   
  + Created Ubuntu Pro account
  + on Beckoff:
    + sudo pro attach [token]
    + sudo pro enable realtime-kernel
    + sudo reboot
    + sudo apt install rt-tests
    + sudo cyclictest
    + check avg and max latency - we had 75us for both, good enough.
    + Modify the netplan:
      + Port 2: enp1s0:
      + Port 1: enp2s0:
      + disable port 2 so that it can be set for the EtherCAT master port

+ Status Checkpoint
   + Ubuntu Server 22.04
   + Ubuntu Pro
   + PREEMPT_RT Kernel
   + Max Latency" 75us
   + Intel I210 NICS x2
   + Management NIC: enp2s0
   + EtherCAT NIC: enp1s0
   
+ IgH EtherCAT Master install
     sudo apt install -y \
    git \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    flex \
    bison \
    libxml2-dev \
    libudev-dev \
    linux-headers-$(uname -r)
     
+ Verify the headers match RT kernerl
      + ls /usr/src/linux-headers-$(uname -r)


 + Clone IgH EtherCAT master
  + cd /usr/src
  + sudo git clone https://gitlab.com/etherlab.org/ethercat.git
  + cd ethercat 

  +  git describe --tags
  +  sudo ./bootstrap
  + ./configure \
    --enable-igb \
    --enable-generic \
    --disable-8139too \
    --disable-e100 \
    --disable-e1000 \
    --disable-r8169

  + make -j$(nproc)
  + make install
  + sudo ldconfig

 + Configure the EtherCAt mASTER
 + sudo nano /etc/ethercat.conf
      MASTER0_DEVICE="enp1s0"
      DEVICE_MODULES="generic"



+Acceptance test: PASS

Service:
  ethercat.service enabled
  ethercat.service active

Modules:
  ec_generic loaded
  ec_master loaded

Master:
  Phase: Idle
  Link: UP
  Slaves: 1
  Lost frames: 0
  Frame loss: 0.0%

Slave 0:
  EK1100 EtherCAT-Koppler
  State: PREOP 



+    Beckhoff C6015
+ │
+ └── Intel I210 (enp1s0)
+    │
+    ▼
+ ┌─────────────┐
+ │ EK1100      │
+ ├─────────────┤
+ │ EL1008      │
+ ├─────────────┤
+ │ EL2008      │
+ └─────────────┘

```
           C6015
                       │
          EtherCAT Master (IgH)
                       │
        ┌──────────────┴──────────────┐
        │                             │
     EK1100                      AL1333
        │                             │
        │                      IO-Link
        │                             │
   EL1008 EL2008              Sensors
                               │
                          Pressure
                          Temp
                          Level
                          Flow
```
SITREP:

+ The PREEMPT_RT kernel is stable.
+ The Intel I210 is operating reliably with the IgH Generic driver.
+ The EtherCAT master starts automatically via systemd.
+ The bus topology is detected correctly after a reboot.
+ The E-bus communication through the EK1100 is functioning.
+ Additional terminals are automatically enumerated.
+ Communication is occurring without packet loss.

Stage 1 Acceptance Criterion: After a complete power cycle, the controller automatically starts the EtherCAT master, binds the Intel I210 interface, enumerates the EK1100, EL1008, and EL2008, reports zero lost frames, and requires no manual intervention.

+ Tomorrows plan:
           
Test Input/Output

Some useful ethernet commands:
systemctl is-enabled ethercat.service
systemctl is-active ethercat.service
lsmod | grep '^ec_'
sudo /usr/local/bin/ethercat master
sudo /usr/local/bin/ethercat slaves

got a permissions issue with my logged in user and the ethercat commands
lets check permissions:
ls -l /dev/EtherCAT0
crw------- 1 root root 235, 0 Aug  3 15:09 /dev/EtherCAT0

gonna create a ethercat group and udev rule
sudo groupadd -f ethercat
sudo usermod -aG ethercat "$USER"
sudo nano /etc/udev/rules.d/99-ethercat.rules
KERNEL=="EtherCAT[0-9]*", GROUP="ethercat", MODE="0660"
sudo udevadm control --reload-rules
sudo systemctl restart ethercat.service

ls -l /dev/EtherCAT0
crw-rw---- 1 root ethercat 235, 0 Aug  3 16:01 /dev/EtherCAT0

reboot
check
groups
ethercat should now show up

now back to checking the EtherCAT system with: EtherCAT bus inspection commands
ethercat slaves -v
ethercat pdos
ethercat cstruct
  
```
C6015 / IgH Master
  └─ Slave 0: EK1100
      └─ Slave 1: EL1008, 8 digital inputs
          └─ Slave 2: EL2008, 8 digital outputs

EL1008: 8 input bits  = 1 byte
EL2008: 8 output bits = 1 byte

EL1008
Channel 1 → 0x6000:01
Channel 2 → 0x6010:01
...
Channel 8 → 0x6070:01

EL2008
Channel 1 → 0x7000:01
Channel 2 → 0x7010:01
...
Channel 8 → 0x7070:01
```

Hello world - sorta
More like, blink this LED.

To light an LED on the 2008. we need a small aplication that can:
+ Requests Master 0.
+ Creates a process-data domain.
+ Registers the EL1008 and EL2008 PDO entries.
+ Activates the master.
+ Runs a cyclic loop.
+ Writes bits into the EL2008 process image.
+ Sends and receives EtherCAT frames.

A small C program that will "Chase" The LEDs on the EL2008, and will also monitor the inputs on the EL1008.
mkdir -p ~/ethercat-io-test
cd ~/ethercat-io-test

ls /usr/local/include/ecrt.h
ls /usr/local/lib/libethercat.so

nano ethercat_io_test.c

```
#define _POSIX_C_SOURCE 200809L

#include <errno.h>
#include <signal.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#include <ecrt.h>

#define MASTER_INDEX 0

#define EK_ALIAS      0
#define EL1008_POS    1
#define EL2008_POS    2

#define BECKHOFF_VENDOR_ID 0x00000002
#define EL1008_PRODUCT     0x03f03052
#define EL2008_PRODUCT     0x07d83052

#define CYCLE_NS 1000000L  /* 1 ms */

static volatile sig_atomic_t keep_running = 1;

static ec_master_t *master = NULL;
static ec_domain_t *domain = NULL;
static uint8_t *domain_pd = NULL;

static unsigned int input_offset[8];
static unsigned int input_bit[8];
static unsigned int output_offset[8];
static unsigned int output_bit[8];

/* EL1008: eight 1-bit input PDOs. */
static ec_pdo_entry_info_t el1008_entries[] = {
    {0x6000, 0x01, 1},
    {0x6010, 0x01, 1},
    {0x6020, 0x01, 1},
    {0x6030, 0x01, 1},
    {0x6040, 0x01, 1},
    {0x6050, 0x01, 1},
    {0x6060, 0x01, 1},
    {0x6070, 0x01, 1},
};

static ec_pdo_info_t el1008_pdos[] = {
    {0x1a00, 1, el1008_entries + 0},
    {0x1a01, 1, el1008_entries + 1},
    {0x1a02, 1, el1008_entries + 2},
    {0x1a03, 1, el1008_entries + 3},
    {0x1a04, 1, el1008_entries + 4},
    {0x1a05, 1, el1008_entries + 5},
    {0x1a06, 1, el1008_entries + 6},
    {0x1a07, 1, el1008_entries + 7},
};

static ec_sync_info_t el1008_syncs[] = {
    {0, EC_DIR_INPUT, 8, el1008_pdos, EC_WD_DISABLE},
    {0xff, EC_DIR_INVALID, 0, NULL, EC_WD_DISABLE}
};

/* EL2008: eight 1-bit output PDOs. */
static ec_pdo_entry_info_t el2008_entries[] = {
    {0x7000, 0x01, 1},
    {0x7010, 0x01, 1},
    {0x7020, 0x01, 1},
    {0x7030, 0x01, 1},
    {0x7040, 0x01, 1},
    {0x7050, 0x01, 1},
    {0x7060, 0x01, 1},
    {0x7070, 0x01, 1},
};

static ec_pdo_info_t el2008_pdos[] = {
    {0x1600, 1, el2008_entries + 0},
    {0x1601, 1, el2008_entries + 1},
    {0x1602, 1, el2008_entries + 2},
    {0x1603, 1, el2008_entries + 3},
    {0x1604, 1, el2008_entries + 4},
    {0x1605, 1, el2008_entries + 5},
    {0x1606, 1, el2008_entries + 6},
    {0x1607, 1, el2008_entries + 7},
};

static ec_sync_info_t el2008_syncs[] = {
    {0, EC_DIR_OUTPUT, 8, el2008_pdos, EC_WD_ENABLE},
    {0xff, EC_DIR_INVALID, 0, NULL, EC_WD_DISABLE}
};

static ec_pdo_entry_reg_t domain_regs[] = {
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6000, 0x01, &input_offset[0], &input_bit[0]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6010, 0x01, &input_offset[1], &input_bit[1]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6020, 0x01, &input_offset[2], &input_bit[2]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6030, 0x01, &input_offset[3], &input_bit[3]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6040, 0x01, &input_offset[4], &input_bit[4]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6050, 0x01, &input_offset[5], &input_bit[5]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6060, 0x01, &input_offset[6], &input_bit[6]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6070, 0x01, &input_offset[7], &input_bit[7]},

    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7000, 0x01, &output_offset[0], &output_bit[0]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7010, 0x01, &output_offset[1], &output_bit[1]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7020, 0x01, &output_offset[2], &output_bit[2]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7030, 0x01, &output_offset[3], &output_bit[3]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7040, 0x01, &output_offset[4], &output_bit[4]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7050, 0x01, &output_offset[5], &output_bit[5]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7060, 0x01, &output_offset[6], &output_bit[6]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7070, 0x01, &output_offset[7], &output_bit[7]},

    {0}
};

static void signal_handler(int signal_number)
{
    (void)signal_number;
    keep_running = 0;
}

static void add_ns(struct timespec *time, long nanoseconds)
{
    time->tv_nsec += nanoseconds;

    while (time->tv_nsec >= 1000000000L) {
        time->tv_nsec -= 1000000000L;
        time->tv_sec++;
    }
}

static void set_all_outputs(int value)
{
    for (int channel = 0; channel < 8; channel++) {
        EC_WRITE_BIT(
            domain_pd + output_offset[channel],
            output_bit[channel],
            value ? 1 : 0
        );
    }
}

static uint8_t read_inputs(void)
{
    uint8_t value = 0;

    for (int channel = 0; channel < 8; channel++) {
        if (EC_READ_BIT(
                domain_pd + input_offset[channel],
                input_bit[channel])) {
            value |= (uint8_t)(1U << channel);
        }
    }

    return value;
}

static int configure_master(void)
{
    ec_slave_config_t *input_config = NULL;
    ec_slave_config_t *output_config = NULL;

    master = ecrt_request_master(MASTER_INDEX);
    if (!master) {
        fprintf(stderr, "Failed to request EtherCAT master 0.\n");
        return -1;
    }

    domain = ecrt_master_create_domain(master);
    if (!domain) {
        fprintf(stderr, "Failed to create process-data domain.\n");
        return -1;
    }

    input_config = ecrt_master_slave_config(
        master,
        EK_ALIAS,
        EL1008_POS,
        BECKHOFF_VENDOR_ID,
        EL1008_PRODUCT
    );

    if (!input_config) {
        fprintf(stderr, "Failed to obtain EL1008 slave configuration.\n");
        return -1;
    }

    output_config = ecrt_master_slave_config(
        master,
        EK_ALIAS,
        EL2008_POS,
        BECKHOFF_VENDOR_ID,
        EL2008_PRODUCT
    );

    if (!output_config) {
        fprintf(stderr, "Failed to obtain EL2008 slave configuration.\n");
        return -1;
    }

    if (ecrt_slave_config_pdos(
            input_config,
            EC_END,
            el1008_syncs)) {
        fprintf(stderr, "Failed to configure EL1008 PDOs.\n");
        return -1;
    }

    if (ecrt_slave_config_pdos(
            output_config,
            EC_END,
            el2008_syncs)) {
        fprintf(stderr, "Failed to configure EL2008 PDOs.\n");
        return -1;
    }

    if (ecrt_domain_reg_pdo_entry_list(domain, domain_regs)) {
        fprintf(stderr, "Failed to register PDO entries.\n");
        return -1;
    }

    if (ecrt_master_activate(master)) {
        fprintf(stderr, "Failed to activate EtherCAT master.\n");
        return -1;
    }

    domain_pd = ecrt_domain_data(domain);
    if (!domain_pd) {
        fprintf(stderr, "Failed to obtain process-data pointer.\n");
        return -1;
    }

    return 0;
}

static void exchange_process_data(void)
{
    ecrt_master_receive(master);
    ecrt_domain_process(domain);

    ecrt_domain_queue(domain);
    ecrt_master_send(master);
}

static void safe_shutdown_outputs(void)
{
    if (!master || !domain || !domain_pd) {
        return;
    }

    set_all_outputs(0);

    /*
     * Send several zero-output cycles so the terminal receives the safe state
     * before the application releases the master.
     */
    for (int i = 0; i < 10; i++) {
        exchange_process_data();

        struct timespec delay = {
            .tv_sec = 0,
            .tv_nsec = CYCLE_NS
        };

        nanosleep(&delay, NULL);
    }
}

static int run_chase(void)
{
    struct timespec wake_time;
    unsigned long cycle = 0;
    int active_channel = -1;

    if (clock_gettime(CLOCK_MONOTONIC, &wake_time) != 0) {
        perror("clock_gettime");
        return -1;
    }

    printf("EL2008 LED chase running. Press Ctrl+C to stop.\n");

    while (keep_running) {
        add_ns(&wake_time, CYCLE_NS);

        ecrt_master_receive(master);
        ecrt_domain_process(domain);

        /*
         * Change channel every 500 cycles = 500 ms at a 1 ms cycle.
         */
        if ((cycle % 500UL) == 0) {
            active_channel = (active_channel + 1) % 8;
            set_all_outputs(0);

            EC_WRITE_BIT(
                domain_pd + output_offset[active_channel],
                output_bit[active_channel],
                1
            );

            printf("\rEL2008 channel %d ON   ",
                   active_channel + 1);
            fflush(stdout);
        }

        ecrt_domain_queue(domain);
        ecrt_master_send(master);

        int result = clock_nanosleep(
            CLOCK_MONOTONIC,
            TIMER_ABSTIME,
            &wake_time,
            NULL
        );

        if (result != 0 && result != EINTR) {
            errno = result;
            perror("clock_nanosleep");
            return -1;
        }

        cycle++;
    }

    printf("\nStopping; forcing all outputs OFF.\n");
    return 0;
}

static int run_monitor(void)
{
    struct timespec wake_time;
    uint8_t previous = 0xff;

    if (clock_gettime(CLOCK_MONOTONIC, &wake_time) != 0) {
        perror("clock_gettime");
        return -1;
    }

    printf("EL1008 input monitor running. Press Ctrl+C to stop.\n");

    while (keep_running) {
        add_ns(&wake_time, CYCLE_NS);

        ecrt_master_receive(master);
        ecrt_domain_process(domain);

        uint8_t current = read_inputs();

        if (current != previous) {
            printf("Inputs: ");

            /*
             * Print channels 8 through 1 so the display resembles a byte.
             */
            for (int bit = 7; bit >= 0; bit--) {
                putchar((current & (1U << bit)) ? '1' : '0');
            }

            printf("  hex=0x%02X\n", current);
            previous = current;
        }

        set_all_outputs(0);
        ecrt_domain_queue(domain);
        ecrt_master_send(master);

        int result = clock_nanosleep(
            CLOCK_MONOTONIC,
            TIMER_ABSTIME,
            &wake_time,
            NULL
        );

        if (result != 0 && result != EINTR) {
            errno = result;
            perror("clock_nanosleep");
            return -1;
        }
    }

    return 0;
}

static void print_usage(const char *program)
{
    fprintf(stderr,
        "Usage: %s chase|monitor\n"
        "\n"
        "  chase    Sequentially light EL2008 output LEDs 1-8.\n"
        "  monitor  Display the EL1008 input byte when it changes.\n",
        program
    );
}

int main(int argc, char **argv)
{
    int result = EXIT_FAILURE;

    if (argc != 2) {
        print_usage(argv[0]);
        return EXIT_FAILURE;
    }

    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);

    if (configure_master() != 0) {
        goto cleanup;
    }

    /*
     * Give the master several seconds of cyclic exchange to move the
     * configured terminals through SAFEOP and into OP.
     */
    printf("Activating EtherCAT process data...\n");

    for (int i = 0; i < 3000 && keep_running; i++) {
        exchange_process_data();

        struct timespec delay = {
            .tv_sec = 0,
            .tv_nsec = CYCLE_NS
        };

        nanosleep(&delay, NULL);
    }

    if (strcmp(argv[1], "chase") == 0) {
        result = run_chase() == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
    } else if (strcmp(argv[1], "monitor") == 0) {
        result = run_monitor() == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
    } else {
        print_usage(argv[0]);
    }

cleanup:
    safe_shutdown_outputs();

    if (master) {
        ecrt_release_master(master);
    }

    return result;
}
```
Ctrl+O
Enter
Ctrl+X

Compile It!
```
gcc \
  -std=c11 \
  -O2 \
  -Wall \
  -Wextra \
  -Wpedantic \
  -I/usr/local/include \
  ethercat_io_test.c \
  -L/usr/local/lib \
  -Wl,-rpath,/usr/local/lib \
  -lethercat \
  -lrt \
  -o ethercat_io_test
```
check it actually exists:
ls -l ethercat_io_test

RUN
sudo ./ethercat_io_test chase

The LEDs on the EL2008 should be chasing.

This validates the full output path:
```
C application
→ IgH userspace API
→ EtherCAT master
→ enp1s0 / Intel I210
→ EK1100
→ EL2008 PDOs
→ physical output LEDs
```
Breadcrumb:
```
2026-08-03 — EL2008 Output Test: PASS

Utility:
  ~/ethercat-io-test/ethercat_io_test

Mode:
  chase

Result:
  Channels 1–8 illuminated sequentially
  Outputs responded correctly
  Ctrl+C returned all outputs OFF

Conclusion:
  Cyclic PDO output communication is functional
  EL2008 output mapping is correct
  EtherCAT application API is working
```

+ Loopback testing
It seems logical to test the outputs and the inputs at the same time to verify all channels are working. It's just a matter of creating a bunch of short jumper wires.
+ Add loopback test capability

```
#define _POSIX_C_SOURCE 200809L

#include <errno.h>
#include <signal.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#include <ecrt.h>

#define MASTER_INDEX 0

#define EK_ALIAS      0
#define EL1008_POS    1
#define EL2008_POS    2

#define BECKHOFF_VENDOR_ID 0x00000002
#define EL1008_PRODUCT     0x03f03052
#define EL2008_PRODUCT     0x07d83052

#define CYCLE_NS 1000000L  /* 1 ms */

static volatile sig_atomic_t keep_running = 1;

static ec_master_t *master = NULL;
static ec_domain_t *domain = NULL;
static uint8_t *domain_pd = NULL;

static unsigned int input_offset[8];
static unsigned int input_bit[8];
static unsigned int output_offset[8];
static unsigned int output_bit[8];

/* EL1008: eight 1-bit input PDOs. */
static ec_pdo_entry_info_t el1008_entries[] = {
    {0x6000, 0x01, 1},
    {0x6010, 0x01, 1},
    {0x6020, 0x01, 1},
    {0x6030, 0x01, 1},
    {0x6040, 0x01, 1},
    {0x6050, 0x01, 1},
    {0x6060, 0x01, 1},
    {0x6070, 0x01, 1},
};

static ec_pdo_info_t el1008_pdos[] = {
    {0x1a00, 1, el1008_entries + 0},
    {0x1a01, 1, el1008_entries + 1},
    {0x1a02, 1, el1008_entries + 2},
    {0x1a03, 1, el1008_entries + 3},
    {0x1a04, 1, el1008_entries + 4},
    {0x1a05, 1, el1008_entries + 5},
    {0x1a06, 1, el1008_entries + 6},
    {0x1a07, 1, el1008_entries + 7},
};

static ec_sync_info_t el1008_syncs[] = {
    {0, EC_DIR_INPUT, 8, el1008_pdos, EC_WD_DISABLE},
    {0xff, EC_DIR_INVALID, 0, NULL, EC_WD_DISABLE}
};

/* EL2008: eight 1-bit output PDOs. */
static ec_pdo_entry_info_t el2008_entries[] = {
    {0x7000, 0x01, 1},
    {0x7010, 0x01, 1},
    {0x7020, 0x01, 1},
    {0x7030, 0x01, 1},
    {0x7040, 0x01, 1},
    {0x7050, 0x01, 1},
    {0x7060, 0x01, 1},
    {0x7070, 0x01, 1},
};

static ec_pdo_info_t el2008_pdos[] = {
    {0x1600, 1, el2008_entries + 0},
    {0x1601, 1, el2008_entries + 1},
    {0x1602, 1, el2008_entries + 2},
    {0x1603, 1, el2008_entries + 3},
    {0x1604, 1, el2008_entries + 4},
    {0x1605, 1, el2008_entries + 5},
    {0x1606, 1, el2008_entries + 6},
    {0x1607, 1, el2008_entries + 7},
};

static ec_sync_info_t el2008_syncs[] = {
    {0, EC_DIR_OUTPUT, 8, el2008_pdos, EC_WD_ENABLE},
    {0xff, EC_DIR_INVALID, 0, NULL, EC_WD_DISABLE}
};

static ec_pdo_entry_reg_t domain_regs[] = {
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6000, 0x01, &input_offset[0], &input_bit[0]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6010, 0x01, &input_offset[1], &input_bit[1]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6020, 0x01, &input_offset[2], &input_bit[2]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6030, 0x01, &input_offset[3], &input_bit[3]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6040, 0x01, &input_offset[4], &input_bit[4]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6050, 0x01, &input_offset[5], &input_bit[5]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6060, 0x01, &input_offset[6], &input_bit[6]},
    {EK_ALIAS, EL1008_POS, BECKHOFF_VENDOR_ID, EL1008_PRODUCT,
     0x6070, 0x01, &input_offset[7], &input_bit[7]},

    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7000, 0x01, &output_offset[0], &output_bit[0]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7010, 0x01, &output_offset[1], &output_bit[1]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7020, 0x01, &output_offset[2], &output_bit[2]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7030, 0x01, &output_offset[3], &output_bit[3]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7040, 0x01, &output_offset[4], &output_bit[4]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7050, 0x01, &output_offset[5], &output_bit[5]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7060, 0x01, &output_offset[6], &output_bit[6]},
    {EK_ALIAS, EL2008_POS, BECKHOFF_VENDOR_ID, EL2008_PRODUCT,
     0x7070, 0x01, &output_offset[7], &output_bit[7]},

    {0}
};

static void signal_handler(int signal_number)
{
    (void)signal_number;
    keep_running = 0;
}

static void add_ns(struct timespec *time, long nanoseconds)
{
    time->tv_nsec += nanoseconds;

    while (time->tv_nsec >= 1000000000L) {
        time->tv_nsec -= 1000000000L;
        time->tv_sec++;
    }
}

static void set_all_outputs(int value)
{
    for (int channel = 0; channel < 8; channel++) {
        EC_WRITE_BIT(
            domain_pd + output_offset[channel],
            output_bit[channel],
            value ? 1 : 0
        );
    }
}

static uint8_t read_inputs(void)
{
    uint8_t value = 0;

    for (int channel = 0; channel < 8; channel++) {
        if (EC_READ_BIT(
                domain_pd + input_offset[channel],
                input_bit[channel])) {
            value |= (uint8_t)(1U << channel);
        }
    }

    return value;
}

static int configure_master(void)
{
    ec_slave_config_t *input_config = NULL;
    ec_slave_config_t *output_config = NULL;

    master = ecrt_request_master(MASTER_INDEX);
    if (!master) {
        fprintf(stderr, "Failed to request EtherCAT master 0.\n");
        return -1;
    }

    domain = ecrt_master_create_domain(master);
    if (!domain) {
        fprintf(stderr, "Failed to create process-data domain.\n");
        return -1;
    }

    input_config = ecrt_master_slave_config(
        master,
        EK_ALIAS,
        EL1008_POS,
        BECKHOFF_VENDOR_ID,
        EL1008_PRODUCT
    );

    if (!input_config) {
        fprintf(stderr, "Failed to obtain EL1008 slave configuration.\n");
        return -1;
    }

    output_config = ecrt_master_slave_config(
        master,
        EK_ALIAS,
        EL2008_POS,
        BECKHOFF_VENDOR_ID,
        EL2008_PRODUCT
    );

    if (!output_config) {
        fprintf(stderr, "Failed to obtain EL2008 slave configuration.\n");
        return -1;
    }

    if (ecrt_slave_config_pdos(
            input_config,
            EC_END,
            el1008_syncs)) {
        fprintf(stderr, "Failed to configure EL1008 PDOs.\n");
        return -1;
    }

    if (ecrt_slave_config_pdos(
            output_config,
            EC_END,
            el2008_syncs)) {
        fprintf(stderr, "Failed to configure EL2008 PDOs.\n");
        return -1;
    }

    if (ecrt_domain_reg_pdo_entry_list(domain, domain_regs)) {
        fprintf(stderr, "Failed to register PDO entries.\n");
        return -1;
    }

    if (ecrt_master_activate(master)) {
        fprintf(stderr, "Failed to activate EtherCAT master.\n");
        return -1;
    }

    domain_pd = ecrt_domain_data(domain);
    if (!domain_pd) {
        fprintf(stderr, "Failed to obtain process-data pointer.\n");
        return -1;
    }

    return 0;
}

static void exchange_process_data(void)
{
    ecrt_master_receive(master);
    ecrt_domain_process(domain);

    ecrt_domain_queue(domain);
    ecrt_master_send(master);
}

static void safe_shutdown_outputs(void)
{
    if (!master || !domain || !domain_pd) {
        return;
    }

    set_all_outputs(0);

    /*
     * Send several zero-output cycles so the terminal receives the safe state
     * before the application releases the master.
     */
    for (int i = 0; i < 10; i++) {
        exchange_process_data();

        struct timespec delay = {
            .tv_sec = 0,
            .tv_nsec = CYCLE_NS
        };

        nanosleep(&delay, NULL);
    }
}

static int run_chase(void)
{
    struct timespec wake_time;
    unsigned long cycle = 0;
    int active_channel = -1;

    if (clock_gettime(CLOCK_MONOTONIC, &wake_time) != 0) {
        perror("clock_gettime");
        return -1;
    }

    printf("EL2008 LED chase running. Press Ctrl+C to stop.\n");

    while (keep_running) {
        add_ns(&wake_time, CYCLE_NS);

        ecrt_master_receive(master);
        ecrt_domain_process(domain);

        /*
         * Change channel every 500 cycles = 500 ms at a 1 ms cycle.
         */
        if ((cycle % 500UL) == 0) {
            active_channel = (active_channel + 1) % 8;
            set_all_outputs(0);

            EC_WRITE_BIT(
                domain_pd + output_offset[active_channel],
                output_bit[active_channel],
                1
            );

            printf("\rEL2008 channel %d ON   ",
                   active_channel + 1);
            fflush(stdout);
        }

        ecrt_domain_queue(domain);
        ecrt_master_send(master);

        int result = clock_nanosleep(
            CLOCK_MONOTONIC,
            TIMER_ABSTIME,
            &wake_time,
            NULL
        );

        if (result != 0 && result != EINTR) {
            errno = result;
            perror("clock_nanosleep");
            return -1;
        }

        cycle++;
    }

    printf("\nStopping; forcing all outputs OFF.\n");
    return 0;
}

static int run_monitor(void)
{
    struct timespec wake_time;
    uint8_t previous = 0xff;

    if (clock_gettime(CLOCK_MONOTONIC, &wake_time) != 0) {
        perror("clock_gettime");
        return -1;
    }

    printf("EL1008 input monitor running. Press Ctrl+C to stop.\n");

    while (keep_running) {
        add_ns(&wake_time, CYCLE_NS);

        ecrt_master_receive(master);
        ecrt_domain_process(domain);

        uint8_t current = read_inputs();

        if (current != previous) {
            printf("Inputs: ");

            /*
             * Print channels 8 through 1 so the display resembles a byte.
             */
            for (int bit = 7; bit >= 0; bit--) {
                putchar((current & (1U << bit)) ? '1' : '0');
            }

            printf("  hex=0x%02X\n", current);
            previous = current;
        }

        set_all_outputs(0);
        ecrt_domain_queue(domain);
        ecrt_master_send(master);

        int result = clock_nanosleep(
            CLOCK_MONOTONIC,
            TIMER_ABSTIME,
            &wake_time,
            NULL
        );

        if (result != 0 && result != EINTR) {
            errno = result;
            perror("clock_nanosleep");
            return -1;
        }
    }

    return 0;
}

static int run_loopback(void)
{
    struct timespec wake_time;
    const unsigned int settle_cycles = 25; /* 25 ms at a 1 ms cycle */
    int passed = 0;

    if (clock_gettime(CLOCK_MONOTONIC, &wake_time) != 0) {
        perror("clock_gettime");
        return -1;
    }

    printf("=== EL2008 to EL1008 Loopback Test ===\n");
    printf("Testing matching channels 1 through 8.\n\n");

    set_all_outputs(0);

    /* Establish an initial all-off state. */
    for (unsigned int cycle = 0; cycle < settle_cycles; cycle++) {
        add_ns(&wake_time, CYCLE_NS);

        ecrt_master_receive(master);
        ecrt_domain_process(domain);

        set_all_outputs(0);

        ecrt_domain_queue(domain);
        ecrt_master_send(master);

        int result = clock_nanosleep(
            CLOCK_MONOTONIC,
            TIMER_ABSTIME,
            &wake_time,
            NULL
        );

        if (result != 0 && result != EINTR) {
            errno = result;
            perror("clock_nanosleep");
            return -1;
        }
    }

    for (int channel = 0; channel < 8 && keep_running; channel++) {
        uint8_t input_state;

        /*
         * Turn on only the output corresponding to the channel under test.
         */
        set_all_outputs(0);

        EC_WRITE_BIT(
            domain_pd + output_offset[channel],
            output_bit[channel],
            1
        );

        /*
         * Continue cyclic communication while allowing the EL1008 input
         * filter and physical signal to settle.
         */
        for (unsigned int cycle = 0;
             cycle < settle_cycles && keep_running;
             cycle++) {

            add_ns(&wake_time, CYCLE_NS);

            ecrt_master_receive(master);
            ecrt_domain_process(domain);

            set_all_outputs(0);

            EC_WRITE_BIT(
                domain_pd + output_offset[channel],
                output_bit[channel],
                1
            );

            ecrt_domain_queue(domain);
            ecrt_master_send(master);

            int result = clock_nanosleep(
                CLOCK_MONOTONIC,
                TIMER_ABSTIME,
                &wake_time,
                NULL
            );

            if (result != 0 && result != EINTR) {
                errno = result;
                perror("clock_nanosleep");
                set_all_outputs(0);
                return -1;
            }
        }

        /*
         * Receive one fresh input sample after the settling period.
         */
        ecrt_master_receive(master);
        ecrt_domain_process(domain);

        input_state = read_inputs();

        const uint8_t expected = (uint8_t)(1U << channel);

        if ((input_state & expected) != 0U) {
            printf(
                "Channel %d: PASS  output=ON input=ON  inputs=0x%02X\n",
                channel + 1,
                input_state
            );
            passed++;
        } else {
            printf(
                "Channel %d: FAIL  output=ON input=OFF inputs=0x%02X\n",
                channel + 1,
                input_state
            );
        }

        /*
         * Turn the tested output off and send the safe state.
         */
        set_all_outputs(0);

        for (unsigned int cycle = 0;
             cycle < settle_cycles && keep_running;
             cycle++) {

            add_ns(&wake_time, CYCLE_NS);

            ecrt_master_receive(master);
            ecrt_domain_process(domain);

            set_all_outputs(0);

            ecrt_domain_queue(domain);
            ecrt_master_send(master);

            int result = clock_nanosleep(
                CLOCK_MONOTONIC,
                TIMER_ABSTIME,
                &wake_time,
                NULL
            );

            if (result != 0 && result != EINTR) {
                errno = result;
                perror("clock_nanosleep");
                return -1;
            }
        }
    }

    set_all_outputs(0);

    printf("\nLoopback result: %d/8 channels passed.\n", passed);

    if (!keep_running) {
        printf("Test interrupted.\n");
        return -1;
    }

    return passed == 8 ? 0 : 1;
}

static void print_usage(const char *program)
{
    fprintf(stderr,
        "Usage: %s chase|monitor|loopback\n"
        "\n"
        "  chase    Sequentially light EL2008 output LEDs 1-8.\n"
        "  monitor  Display the EL1008 input byte when it changes.\n"
        "  loopback Test EL2008 outputs against matching EL1008 inputs.\n",
        program
    );
}

int main(int argc, char **argv)
{
    int result = EXIT_FAILURE;

    if (argc != 2) {
        print_usage(argv[0]);
        return EXIT_FAILURE;
    }

    signal(SIGINT, signal_handler);
    signal(SIGTERM, signal_handler);

    if (configure_master() != 0) {
        goto cleanup;
    }

    /*
     * Give the master several seconds of cyclic exchange to move the
     * configured terminals through SAFEOP and into OP.
     */
    printf("Activating EtherCAT process data...\n");

    for (int i = 0; i < 3000 && keep_running; i++) {
        exchange_process_data();

        struct timespec delay = {
            .tv_sec = 0,
            .tv_nsec = CYCLE_NS
        };

        nanosleep(&delay, NULL);
    }

    if (strcmp(argv[1], "chase") == 0) {
    result = run_chase() == 0
        ? EXIT_SUCCESS
        : EXIT_FAILURE;

} else if (strcmp(argv[1], "monitor") == 0) {
    result = run_monitor() == 0
        ? EXIT_SUCCESS
        : EXIT_FAILURE;

} else if (strcmp(argv[1], "loopback") == 0) {
    result = run_loopback() == 0
        ? EXIT_SUCCESS
        : EXIT_FAILURE;

} else {
    print_usage(argv[0]);
}

cleanup:
    safe_shutdown_outputs();

    if (master) {
        ecrt_release_master(master);
    }

    return result;
}
```
+ That works pretty good. It will sequentially test east output/input pair. so when a jumper is present it will verify both modules are functioning.

+  Digital I/O Commissioning: PASS
+  ```
                  Software
                   │
                   ▼
      IgH EtherCAT Application
                   │
                   ▼
          EtherCAT Master
                   │
                   ▼
             Intel I210 NIC
                   │
                   ▼
               EK1100 Coupler
              ┌─────────────┐
              │             │
          EL2008        EL1008
              │             ▲
              └────Wire─────┘
   ```
   
+ Next steps:
Connect the IFM AL1333 to the EtherCAT fieldbus, and test that connection, then add in some IO link devices for testing. The goal is to determine if its best to use something like codesys which is only a "soft" real time, or program our own realtime scheduler in C, then add the required "modules" to complete out the system. 

+ Pause for some system integration
Wired the M12 for the IFM AL1333 power, and connected the M12 12thernet port to port 2 on the ek1100
Installed one IO-Link sensor, a PT100 probe.

```
                    STEAM OUT
                        │
               ┌────────┴────────┐
               │                 │
               │   PM1704        │  Vessel pressure
               │                 │
               │   TD2531 or     │  Steam/water temperature
               │   TR2439+RTD    │
               │                 │
               │   LR2750        │  Continuous water level
               │                 │
               │   LMT100        │  Independent low-level trip
               │                 │
               └───────┬─────────┘
                       │
                    HEAT INPUT
```
```
X01 — PM1704 pressure
X02 — LR2750 continuous level
X03 — LMT100 low-water switch
X04 — TR2439 temperature
X05 — TD2531 temperature
X06–X08 — spare
```

# Where we're going, we don't need CODESYS
+ So now to develop a light weight control system from thin air. Will continue leveraging the IgH EtherCAT software, and the previous integrations. Up to this point it was core system set up, initialize EtherCAT, and test interfaces. All have passed their checks, so now its time to move on to the next piece of software.

+ High level overview, build a simple runtime that:
  + Starts as a systemd service
  + requests IgH master 0
  + Confiogures the know bus
  + Runs cyclic tasks
  + Exposes a normalized process model
  + Forces all outputs safe on shutdown, communications loss, or other detected fault.
 
```
typedef struct {
    struct {
        bool inputs[8];
        bool outputs[8];
    } cabinet_io;

    struct {
        float temperature_c;
        float pressure_bar;
        float level_mm;
        bool low_level;
        bool valid;
    } steam_generator;
} process_image_t;
```

+ State Machine Plan
```
OFF
→ FILLING
→ READY
→ HEATING
→ PRESSURE_CONTROL
→ SHUTDOWN
→ FAULT_LOCKOUT
```
