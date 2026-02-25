# **Architectural Analysis of USB/IP, Hardware Virtualization, and Peripheral Integration in Non-Rooted Android PRoot Environments**

## **Introduction to the Android Virtualization Paradigm and Hardware Constraints**

The execution of native Linux distributions on Android hardware without root privileges represents a complex intersection of user-space virtualization, system call interception, and strict sandboxing paradigms. Through environments like Termux and its associated proot-distro utility, it is entirely possible to deploy fully functioning Linux user-spaces—such as Ubuntu, Debian, Alpine Linux, or Arch Linux—directly on an Android kernel.1 However, a fundamental architectural limitation arises when attempting to bridge external hardware peripherals, such as USB flash drives, serial adapters, or micro-controller programmers, directly into these chroot-like environments.

The core inquiry of modern mobile virtualization revolves around whether a usbipd (USB/IP daemon) server implementation exists for Termux PRoot on a non-rooted Android device, and specifically, whether this protocol can successfully mount physical hardware into the PRoot Ubuntu environment. Addressing this requires an exhaustive deconstruction of the Android USB subsystem, the limitations of ptrace-based system call interception, the architecture of the USB/IP protocol, and the dichotomy between user-space libusb backends and kernel-space virtual host controller interfaces (vhci-hcd).

The analysis indicates that while user-space USB/IP servers (implementing usbipd) do indeed exist and can successfully bind to Android USB devices without root permissions via file descriptor handover, the receiving end—the PRoot Ubuntu environment acting as the USB/IP client—cannot natively mount these devices as generic block or character nodes due to the strict absence of the vhci-hcd kernel module.2 Consequently, connecting hardware to a PRoot Ubuntu instance necessitates specialized architectural workarounds. These bypass mechanisms range from patching the libusb dependencies of target applications using Inter-Process Communication (IPC) bridges, to leveraging full-system emulators like QEMU to instantiate an independent kernel space capable of parsing USB request blocks.

## **The Architecture of Termux and PRoot System Call Interception**

To understand the barrier to hardware access, one must first analyze the precise mechanical execution of PRoot. Unlike traditional virtual machines that run an independent guest kernel, or fully privileged Linux containers (LXC/Docker) that utilize kernel namespaces and control groups (cgroups), PRoot is strictly a user-space implementation of chroot, mount \--bind, and binfmt\_misc.

The functional foundation of PRoot relies on the ptrace system call to intercept, suspend, and manipulate the system calls made by the guest processes (e.g., Ubuntu binaries) before they are processed by the host Android kernel.1 When a process inside the PRoot Ubuntu environment attempts to access a file or a device node, PRoot intercepts the open, openat, or faccessat system call, translates the requested file path to the corresponding localized path within the isolated root filesystem (rootfs), and passes the modified system call back to the kernel for execution.1

The \-0 flag in the standard proot execution string is specifically designed to simulate a root user environment. It achieves this by intercepting calls like getuid, geteuid, and chown, forcefully returning 0 (the standard Unix identifier for the root user) to the calling application. This sophisticated masquerade tricks package managers like apt or pacman into functioning under the illusion of total system control. However, this is strictly a user-space illusion; PRoot does not provide any mechanism for real privilege escalation. Programs requiring real root access to modify kernel states, alter hardware configurations, insert kernel modules, or mount external file systems will invariably fail because the host Android kernel recognizes the true, unprivileged UID of the Termux application.1

### **The /dev Filesystem Binding and Hardware Blindness**

In a standard Linux operating system, physical hardware device nodes reside within the dynamically populated /dev directory. In a PRoot deployment, the host Android device's /dev, /proc, and /sys filesystems are synthetically bound to the guest environment using the \-b /dev \-b /proc \-b /sys flags. Despite this binding, Android's strict Security-Enhanced Linux (SELinux) policies and mandatory access control (MAC) user-group permissions prevent a non-root application from directly interacting with raw device nodes, such as /dev/bus/usb/001/002.6

Furthermore, standard Linux initialization systems and hardware management daemons, such as systemd and udev, suffer catastrophic failures when executed inside PRoot. Because Android does not utilize systemd as its initialization daemon, the PRoot Ubuntu instance boots without a functional PID 1 systemd process.8 Attempts to interact with system services return fatal errors indicating that the host bus is down.8 More critically, the kernel readily recognizes the chroot environment and refuses to launch the udev daemon, outputting explicit warnings such as "A chroot environment has been detected, udev not started".8 Without udev listening to kernel uevent netlink sockets, automatic hardware enumeration, driver binding, and dynamic device node creation are entirely broken inside the Ubuntu guest.8

## **Android USB Subsystem and File Descriptor Handover Mechanics**

Because direct, kernel-level interaction with /dev/bus/usb is administratively denied to non-root applications, the Android operating system enforces a strict, Java-based permission model for hardware access. An application must interface with the Android UsbManager API to request explicit, user-facing permission to access a specific connected peripheral. Upon the user granting this permission via a system UI dialog, the Android OS bypasses the traditional /dev pathing and returns an open, authenticated file descriptor (FD) pointing directly to the device's memory space.6

### **The termux-usb Utility and IPC Execution**

The Termux ecosystem navigates this severe restriction via the auxiliary Termux:API application and the associated termux-usb command-line utility. This utility facilitates a highly orchestrated, three-step protocol to expose physical USB hardware to command-line Linux software running within the terminal.6

The first phase involves enumeration, where executing termux-usb \-l lists the available connected devices that are recognized by the underlying Android OS.6 The second phase is the permission request; executing termux-usb \-r /dev/bus/usb/001/002 triggers an IPC call to the Java API, rendering a permission dialog on the device screen.6 The final, most critical phase is the execution and handover. By executing termux-usb \-e \<command\> /dev/bus/usb/001/002, the utility launches the specified command or binary as a child process, passing the authenticated file descriptor as a command-line argument.6

### **The libusb Abstraction Gap and C-Level Initialization**

Most standard open-source Linux applications that interact with peripheral hardware (e.g., lsusb, avrdude for microcontroller flashing, sane for scanning) rely on the foundational libusb library to abstract hardware communication. Traditionally, libusb relies on recursively scanning the /dev/bus/usb directory tree and handling the open() system calls natively to acquire file descriptors. Because Termux passes an already-opened file descriptor as an integer rather than a resolvable path, the native, unmodified libusb binaries shipped inside the PRoot Ubuntu environment cannot automatically discover, enumerate, or utilize the attached device.6

To bridge this specific abstraction gap, software must be explicitly written or heavily patched to accept the file descriptor via standard input or execution arguments, and subsequently initialize the device using the highly specific libusb\_wrap\_sys\_device() C function.6 The fundamental code structure for this operation requires a precise sequence of library calls.

| Initialization Step | libusb Function Call | Architectural Purpose |
| :---- | :---- | :---- |
| **1\. FD Ingestion** | sscanf(argv, "%d", \&fd) | Parses the integer representation of the file descriptor passed by the termux-usb \-e wrapper.6 |
| **2\. Discovery Suppression** | libusb\_set\_option(NULL, LIBUSB\_OPTION\_NO\_DEVICE\_DISCOVERY) | Crucial for Android PRoot environments. Prevents the library from crashing or returning permission errors when attempting to scan the inaccessible /dev tree.6 |
| **3\. Context Initialization** | libusb\_init(\&context) | Initializes the baseline libusb session context.6 |
| **4\. Device Wrapping** | libusb\_wrap\_sys\_device(context, (intptr\_t) fd, \&handle) | Forcefully wraps the Android-provided, authenticated file descriptor into a standardized libusb\_device\_handle, allowing standard URB transfers.6 |

While this specialized mechanism successfully allows custom C programs or sophisticated Python scripts (via patched pyusb integrations) to read and write USB Request Blocks (URBs) directly to the hardware 10, it fundamentally fails to provide a seamless "plug-and-play" experience. Standard, pre-compiled Ubuntu binaries acquired via apt-get remain entirely blind to the hardware, as they lack the custom logic to ingest external file descriptors.12

## **The USB/IP Protocol Architecture: Standard Implementations**

To circumvent the exhaustive necessity of recompiling and patching every individual user-space application to support file descriptor ingestion, system architects frequently turn to the USB/IP protocol. The USB/IP project is a generalized USB device sharing system operating over an IP network. It encapsulates low-level USB I/O messages—specifically USB Request Blocks (URBs)—into TCP/IP payloads, transmitting them between computers. This architecture allows a client machine to utilize remote USB devices as though they were physically attached to the local PCI bus.13

### **The Kernel-Space Dichotomy**

In a standard, fully privileged Linux environment, the USB/IP framework operates symmetrically within the kernel space, strictly divided into Server (Host) and Client (Guest) architectures.14 The server machine, which physically hosts the USB device, requires the usbip-core.ko and usbip-host.ko kernel modules. The stub driver intercepts URBs directly from the physical USB device controller, preventing local drivers from claiming the device, and queues the data for network transmission.4

Conversely, the client machine requires usbip-core.ko and the Virtual Host Controller Interface module, vhci-hcd.ko. The vhci-hcd module is an intricate piece of kernel engineering; it emulates a physical USB host controller on the motherboard. It receives the incoming TCP payloads, decapsulates the URBs, and injects them seamlessly into the local kernel's USB subsystem. To the client operating system's driver stack, the decapsulated payload appears identical to an interrupt or bulk transfer from a physical port.4

The user-space daemon, usbipd, handles the initial TCP handshakes, device enumeration, and access control, but it relies entirely on these underlying kernel modules to perform the actual, high-speed data multiplexing and device node creation.14

## **User-Space USB/IP Servers (usbipd) on Android**

Addressing the specific user query: a usbipd server implementation *does* exist for non-rooted Termux environments. Because Android restricts access to the kernel-space usbip-host.ko module, the open-source community has engineered sophisticated user-space implementations of the USB/IP protocol that interact directly with the libusb framework, thereby bypassing the kernel stub driver requirement entirely.4

### **The usbipdcpp Asynchronous Architecture**

A highly prominent and technically mature solution for non-rooted Android environments is usbipdcpp, a C++ library engineered specifically to instantiate a USB/IP server entirely in user-space.16 This implementation leverages the libusb asynchronous API in conjunction with the file descriptor handed over by the termux-usb utility to broadcast the physical hardware.

The internal architecture of usbipdcpp is designed to mitigate the heavy resource constraints associated with simultaneous network I/O and continuous USB polling. It implements a fully asynchronous architecture utilizing C++20 coroutines and the asio library.16 To prevent synchronization operations on specific devices from blocking the global execution state, the threading model is explicitly segregated into three dedicated execution loops 16:

1. **Network I/O Thread:** Executes asio::io\_context::run(), autonomously waiting for incoming client TCP connections, handling the USB/IP handshake, and encapsulating outgoing URBs into IP payloads without blocking.16  
2. **USB Transfer Thread:** Dedicated exclusively to managing libusb\_handle\_events(). This ensures that high-speed asynchronous callbacks from the physical hardware (such as isochronous streams from webcams or bulk transfers from mass storage) are processed immediately.16  
3. **Main Control Thread:** Manages the overarching lifecycle, configuration, and teardown of the server instance.16

By executing the server via the Termux wrapper—specifically termux-usb \-e /path/to/termux\_libusb\_server /dev/bus/usb/001/002—the physical device attached to the Android OTG port is successfully converted into a network-accessible USB/IP peripheral.16 Furthermore, because the termux-usb utility limits execution to a single file descriptor per invocation, developers can instantiate multiple parallel server instances on varying network ports (defined via the USBIPDCPP\_LISTEN\_PORT environment variable) to broadcast multiple connected devices simultaneously.16

### **Rust-Based Implementations and Alternative Backends**

In parallel to C++ implementations, the Rust ecosystem has produced highly resilient user-space USB/IP servers. Projects such as jiegec/usbip provide a comprehensive Rust library capable of running a USB/IP server backed entirely by libusb.4 This allows the Rust implementation to function dynamically as a USB driver on any platform that restricts kernel module compilation, seamlessly supporting macOS, BSD, and non-rooted Android architectures.4

These implementations firmly establish that an Android device can act as a fully functional USB/IP Host without root privileges. A separate physical machine, such as a Windows 11 desktop running dorssel/usbipd-win or a standard Linux server equipped with vhci-hcd, can connect to the Android IP address, execute the attachment protocol, and utilize the mobile peripheral.20 However, while this solves the server-side equation, it does not intrinsically solve the core user requirement: connecting the hardware *inward* to the local PRoot Ubuntu environment running on the same Android device.

## **The Client-Side Deficit: Why PRoot Ubuntu Cannot Mount Devices Natively**

If physical hardware is being exposed by a user-space USB/IP server (like usbipdcpp) broadcasting on the Android localhost loopback interface (127.0.0.1), the PRoot Ubuntu instance must assume the role of the USB/IP Client to ingest it. As established, standard Linux USB/IP client operations execute the usbip attach command, which communicates strictly with the vhci-hcd kernel module to instantiate the virtual hardware node.14

Because the PRoot user-space shares the underlying, unmodified Android kernel, this standard attachment process encounters a fatal structural blockade. Android OS kernels deployed by Original Equipment Manufacturers (OEMs) do not compile or ship the vhci-hcd.ko module by default.24 Furthermore, non-rooted users explicitly lack the CAP\_SYS\_MODULE Linux capability required to insert custom kernel modules via insmod or modprobe, even if an engineer managed to accurately cross-compile the module against the specific OEM kernel headers.26

Thus, executing a standard sudo modprobe vhci-hcd inside the PRoot Ubuntu terminal will instantly fail with a "Module vhci-hcd not found" or permission denied error.26 Without the virtualization capabilities of vhci-hcd, the Ubuntu OS possesses no mechanism to interpret the TCP payload and create the virtual /dev nodes necessary to expose the hardware to standard applications.29

### **The Limitations of User-Space USB/IP Clients**

To bypass the strict vhci-hcd kernel requirement, security researchers and developers have attempted to engineer user-space USB/IP clients. Projects such as forensix/libusbip represent a C library built atop libusb designed to act as a network client.4 Additionally, sophisticated abstraction layers like VIIPER seek to eliminate kernel programming entirely by handling USB device emulation directly in user-space code.30

However, a fundamental operating system constraint limits the utility of these user-space clients inside a generalized PRoot environment. A user-space client library can successfully establish the TCP handshake with the usbipdcpp server, decapsulate the incoming URB payloads, and parse the raw hexadecimal data.32 What it fundamentally *cannot* do is instruct the Android kernel to register a generic block storage device (e.g., /dev/sdb for a USB drive) or a character device (e.g., /dev/ttyUSB0 for a serial adapter).3

The creation of generic device nodes that transparently interface with existing, compiled software (like the fdisk partitioning tool, the mount command, or serial terminal emulators) requires the initialization of kernel-level drivers (e.g., usb-storage or cdc-acm).3 Because standard Ubuntu applications implicitly expect these device nodes to exist in the filesystem hierarchy, a user-space USB/IP client is practically ineffective for general-purpose desktop use. The only applications capable of utilizing a user-space USB/IP client would be those specifically rewritten and recompiled to link against the custom libusbip API instead of issuing standard POSIX open() calls to the /dev filesystem—a monumental undertaking that defeats the primary purpose of running a standard, pre-packaged Ubuntu distribution.4

### **Security Context: Why Android Restricts vhci-hcd**

The inability to load vhci-hcd into the Android kernel, while highly restrictive for virtualization deployments, is a deliberate architectural safeguard designed to protect the mobile device from established vulnerability classes. The vhci-hcd module has historically suffered from memory corruption bugs, pointer mishandling, and severe information leaks.

Recent vulnerability disclosures, such as CVE-2024-43883 and CVE-2024-39499, highlight critical flaws where the vhci-hcd driver drops internal references prematurely or fails to sanitize index inputs originating from user-space.33 These vulnerabilities allow malicious actors to exploit stale pointers, potentially leading to Local Privilege Escalation (LPE) out of a sandboxed container.33 Furthermore, hardware testing indicates that extreme bulk IN URB sizes (exceeding 20 KiB) transmitted over vhci\_hcd on ARM64 architectures can trigger kernel oops events and unrecoverable system panics.34 By restricting arbitrary kernel module insertion and forcing all USB interactions through the highly regulated Java UsbManager and termux-usb file descriptor handovers, the Android OS effectively isolates its core execution ring from malformed USB payloads originating from experimental PRoot environments.

## **Architectural Workarounds for Hardware Access in PRoot Ubuntu**

Given that standard USB/IP client operations fail due to the unresolvable missing vhci-hcd module, systems engineers have developed two distinct, highly technical pathways to achieve hardware connectivity in non-rooted Termux PRoot environments. These pathways abandon standard USB/IP client tools in favor of library manipulation and full-system emulation.

### **Workaround Methodology I: Library Patching via AnotherTerm**

A highly advanced, albeit fragile, methodology involves replacing the standard libusb shared object (libusb-1.0.so.0) within the PRoot Linux filesystem with a heavily modified variant. This approach was pioneered by the AnotherTerm application ecosystem, which provides a custom-patched libusb designed to integrate directly with a dedicated helper service running natively in the Android host layer.36

When an unmodified Ubuntu application executing inside PRoot (such as OpenOCD for micro-controller debugging) invokes standard libusb discovery functions, the patched library intercepts these internal function calls.36 Instead of attempting to scan the restricted /dev/bus/usb directory tree, the patched library communicates via local Unix domain sockets to the Android helper service. The helper service programmatically triggers the Android system UI to request user permission, retrieves the authenticated file descriptor from the Android USB host subsystem, and passes it back to the patched libusb inside PRoot via Inter-Process Communication (IPC).37

To implement this specialized bridge inside a standard Termux PRoot Ubuntu installation, users must rely on custom compilation scripts, specifically install-patched-libusb-1.0.26.sh, provided by the AnotherTerm-scripts repository.36 Executing this script within the PRoot environment downloads, compiles, and forcefully overwrites the Ubuntu distribution's default libusb libraries.36

#### **Capability Analysis of Library Patching**

| Architectural Feature | Operational Impact |
| :---- | :---- |
| **Application Transparency** | Ubuntu applications relying strictly on libusb for hardware interaction require zero source-code modification. They remain unaware that their library calls are being routed through Android IPC.36 |
| **Execution Performance** | By utilizing local Unix domain sockets for file descriptor handover, this method avoids the heavy latency and processing overhead associated with the TCP/IP encapsulation required by the USB/IP protocol.37 |
| **System Stability Constraints** | The patch deployment is inherently fragile. Routine package manager updates (e.g., executing apt upgrade) may overwrite the custom-patched library with the upstream Ubuntu version, instantly breaking hardware access and requiring manual script re-execution.8 |
| **Scope Limitation (Critical)** | This workaround is strictly limited to devices managed exclusively via libusb (e.g., smart card readers, SDR dongles, specialized programmers). It **does not** enable block storage mounting, generic serial ports, or UVC webcams, as these require full kernel-level driver initialization which libusb cannot simulate.12 |

The empirical evidence strongly suggests that while patching libusb allows interaction with specific peripheral classes, it fails as a generalized solution for hardware access. As explicitly confirmed by Termux project maintainers, programs requiring access to external mass storage drives cannot function via this method. Interfacing with storage requires block device registration and Filesystem in Userspace (FUSE) mounting capabilities, neither of which are permitted by the Android kernel for PRoot-managed processes.2

### **Workaround Methodology II: Emulated Kernel Injection via usbredirect and QEMU**

The most robust, generalized, and fully capable solution to the non-rooted hardware limitation abandons the PRoot execution layer entirely for hardware-dependent tasks, pivoting instead to full system emulation via QEMU. While QEMU is computationally heavier than the lightweight ptrace hooking utilized by PRoot, it instantiates its own fully independent, virtualized Linux kernel, thereby bypassing Android's restrictions entirely.41

The architecture of this setup mirrors the network philosophy of USB/IP but relies on QEMU's internal Character Device (chardev) mechanisms rather than vhci-hcd. The data pipeline relies on cross-compiling a tool called usbredirect (derived from the usbredir library ecosystem) and launching it via the termux-usb wrapper.43

#### **The usbredirect Execution Pipeline**

The connection pipeline is established through a precise sequence of network and descriptor hand-offs, requiring strict adherence to command-line syntax:

1. **Host-Side File Descriptor Binding:** The user executes termux-usb \-e wrapped around the usbredirect binary in the native Termux shell (outside of PRoot). A critical syntax must be maintained to prevent Termux argument parsing errors. The execution string must encapsulate the binary and its flags within a single quote block: termux-usb \-e "/path/to/usbredirect \--device /dev/bus/usb/001/031 \--as 127.0.0.1:23456" /dev/bus/usb/001/031.43  
2. **TCP Socket Initialization:** The termux-usb wrapper elevates the request to the Android OS, obtains the file descriptor, and launches usbredirect. The usbredirect application intercepts the raw USB endpoint data and binds it to a local TCP socket (127.0.0.1:23456).43  
3. **QEMU Monitor Attachment:** Inside the Termux environment, a QEMU virtual machine is launched (e.g., booting an Alpine Linux or Debian image). Through the QEMU control monitor interface (often accessed via a Unix socket or directly via execution arguments), the exposed TCP socket is bridged to a virtual character device: chardev-add socket,host=127.0.0.1,port=23456,id=c1.43  
4. **Virtual Hardware Injection:** Finally, the newly created virtual character device is bound to QEMU's emulated USB controller, injecting the data stream into the guest OS: device\_add usb-redir,chardev=c1,id=u1,debug=3.43

#### **Analyzing the QEMU Virtualization Advantage**

The undeniable success of the usbredirect pipeline highlights the functional disparity between PRoot and QEMU. When the data stream enters the QEMU environment via the usbredir protocol, it interfaces directly with the guest operating system's kernel.41 Because the guest OS (e.g., Debian running inside QEMU) possesses a fully operational kernel equipped with a comprehensive suite of standard drivers (like usb-storage), the guest kernel successfully interprets the URBs, triggering its internal udev daemon to create the required /dev nodes.43

This architecture allows standard Linux operations that are impossible in PRoot to execute seamlessly. Technical documentation demonstrates that once a physical USB flash drive connected to an unrooted Android phone is attached to QEMU via this method, it can be formatted natively using mkfs.ext4, partitioned utilizing fdisk, and even subjected to full-disk Linux Unified Key Setup (LUKS) encryption via cryptsetup.43 Because the QEMU guest possesses an authoritative /dev tree, the hardware is treated identically to local, physical hardware attached to a standard motherboard.43

While this methodology solves the hardware access issue comprehensively—allowing for generic block storage and character device manipulation—the cost is severe execution performance degradation. QEMU running inside Termux on an ARM architecture often relies on Tiny Code Generator (TCG) dynamic binary translation or generic software virtualization, which incurs significant computational overhead compared to the near-native execution speed of PRoot.48

## **Alternative Bridging Techniques and Edge-Case Limitations**

For developers insisting on maintaining the high-performance execution profile of PRoot Ubuntu while circumventing the complex libusb patching methodologies, highly specific edge-case bridging technologies have been explored. However, these solutions expose the rigid limitations of user-space abstractions.

### **Socat Relays for Serial Telemetry**

For highly specific, low-bandwidth peripherals, such as USB-to-Serial adapters (e.g., CH340 or FTDI interface chips), researchers have successfully utilized the socat utility to relay data.49 Because PRoot restricts direct access to /dev/ttyUSB0, a script running in the native Termux environment can interact with the serial device using the Android Java API (often via packages like termux-usb), and subsequently use socat to bind the serial output stream to a User Datagram Protocol (UDP) port or a Unix domain socket located in a shared directory (e.g., /tmp/ttyUSB\_relay).49

The PRoot Ubuntu instance can then read and write to this localized socket. While highly functional for simple serial telemetry and text-based modem communication, this approach breaks down entirely for complex protocols. Serial data is a relatively simple, continuous byte stream, whereas hardware like UVC webcams or Wi-Fi adapters rely on highly structured, timing-sensitive isochronous and bulk transfers that simple socat pipes cannot manage or synchronize.39

### **Python Abstractors and SELinux Denials**

Applications written in high-level interpreted languages like Python face identical constraints inside PRoot. The widely used pyusb library natively calls the underlying C-based libusb architecture. Experimental proposals and pull requests within the pyusb ecosystem (such as PR \#287) have attempted to implement custom open helpers configured via environment variables to ingest the Android file descriptor dynamically.11

The theoretical workflow allows a Python script executing inside PRoot to call termux-usb externally, which then executes a secondary script that passes the acquired file descriptor back via a Unix pipe.38 The Python script then manually constructs a libusb\_context using libusb\_wrap\_sys\_device. While technically elegant, Android's SELinux policies strictly monitor and frequently terminate cross-process file descriptor sharing between untrusted contexts. Attempts to use the proot command-line utility to bind /dev/bus/usb/ to a temporary directory with symlinks mapped to /proc/self/fd/ often trigger immediate SELinux denials. These denials manifest as unexpected application crashes, segmentation faults, or cryptic "Unknown error \-14" responses during ptrace intercept operations, rendering the abstraction highly unstable for production workflows.5

## **Strategic Conclusions and Future Outlook**

The integration of physical hardware into non-rooted Android PRoot environments represents a profound architectural clash between Android's highly regulated, Java-abstracted hardware permission model and the traditional Linux ecosystem's expectation of direct, kernel-mediated access to character and block device nodes.

An exhaustive analysis of the usbip protocol, the Android /dev filesystem constraints, and the user-space emulation layer yields the following definitive determinations regarding hardware connectivity in PRoot Ubuntu:

1. **The Existence of Userspace usbipd Servers:** Implementations of the USB/IP server daemon (usbipd) absolutely exist for non-rooted Termux environments. Libraries such as usbipdcpp (C++) and jiegec/usbip (Rust) successfully utilize the file descriptor handed over by termux-usb to broadcast physical Android peripherals over a TCP/IP network layer.4 Therefore, the Android device can reliably act as a hardware host for external network clients.  
2. **The Failure of the USB/IP Client Inside PRoot:** While the server aspect is technologically resolved, attempting to mount this broadcasted hardware *inward* to the local PRoot Ubuntu instance fails unequivocally. Standard USB/IP clients rely intrinsically on the vhci-hcd kernel module to establish virtual hardware nodes within the /dev filesystem.14 Because PRoot shares the unmodified Android kernel—which lacks vhci-hcd and vehemently denies module insertion privileges (CAP\_SYS\_MODULE)—the client attachment process is functionally impossible.2  
3. **The Limitations of Library Patching:** For highly specific use cases involving applications that rely purely on libusb (such as SDRs or micro-controller programmers), the PRoot environment's default libusb can be overwritten with custom-patched versions (e.g., via AnotherTerm-scripts). These patches route hardware discovery through Android's IPC mechanisms, bypassing the kernel.36 However, this is a fragile, application-specific workaround that completely fails for generic block storage, audio devices, or network adapters that require kernel driver initialization.12  
4. **The Absolute Viability of Emulation Workarounds:** The only comprehensive method for exposing raw hardware to a Linux environment on a non-rooted Android device is to transition from PRoot to full system emulation. By executing usbredirect wrapped in termux-usb, the raw USB datastream is pushed over a local TCP socket. A QEMU virtual machine running inside Termux can then ingest this socket via the usbredir protocol, allowing its fully independent guest kernel to load native drivers, format partitions, and mount the hardware seamlessly.43

Ultimately, while usbipd servers exist for Termux, the USB/IP protocol is inherently unsuited for injecting hardware *into* a local PRoot environment due to strict kernel-space requirements on the receiving client side. Systems engineers attempting to construct development, penetration testing, or hardware reverse-engineering environments on non-rooted Android devices must choose between the high performance but hardware-blind execution of PRoot, or the computationally expensive but hardware-capable emulation of QEMU.

#### **Works cited**

1. PRoot \- Termux Wiki, accessed on February 24, 2026, [https://wiki.termux.com/wiki/PRoot](https://wiki.termux.com/wiki/PRoot)  
2. termux/proot-distro: An utility for managing installations of the Linux distributions in Termux. \- GitHub, accessed on February 24, 2026, [https://github.com/termux/proot-distro](https://github.com/termux/proot-distro)  
3. USB/IP protocol support \- Technical Documentation \- Nordic Semiconductor, accessed on February 24, 2026, [https://docs.nordicsemi.com/bundle/ncs-3.1.0/page/zephyr/connectivity/usb/host/usbip.html](https://docs.nordicsemi.com/bundle/ncs-3.1.0/page/zephyr/connectivity/usb/host/usbip.html)  
4. usbip/implementations: USB/IP Server/Client/Userspace Implementations \- GitHub, accessed on February 24, 2026, [https://github.com/usbip/implementations](https://github.com/usbip/implementations)  
5. proot warning: ptrace(POKEDATA): I/O error · Issue \#15 \- GitHub, accessed on February 24, 2026, [https://github.com/termux/proot/issues/15](https://github.com/termux/proot/issues/15)  
6. Termux-usb, accessed on February 24, 2026, [https://wiki.termux.com/wiki/Termux-usb](https://wiki.termux.com/wiki/Termux-usb)  
7. how to mount SD/USB in Termux PRoot-Distro? \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/17joddh/how\_to\_mount\_sdusb\_in\_termux\_prootdistro/](https://www.reddit.com/r/termux/comments/17joddh/how_to_mount_sdusb_in_termux_prootdistro/)  
8. ELI5 : What are the limitations and problems of running Ubuntu on Termux proot (non-rooted device)? \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/ho16du/eli5\_what\_are\_the\_limitations\_and\_problems\_of/](https://www.reddit.com/r/termux/comments/ho16du/eli5_what_are_the_limitations_and_problems_of/)  
9. ChangeLog-5.4.7 \- The Linux Kernel Archives, accessed on February 24, 2026, [https://cdn.kernel.org/pub/linux/kernel/v5.x/ChangeLog-5.4.7](https://cdn.kernel.org/pub/linux/kernel/v5.x/ChangeLog-5.4.7)  
10. Any luck writing to USB device? \- termux \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/18w4rid/any\_luck\_writing\_to\_usb\_device/](https://www.reddit.com/r/termux/comments/18w4rid/any_luck_writing_to_usb_device/)  
11. Termux (Android support), usb device from sys device (file descriptor), Issue \#285 by Querela · Pull Request \#287 \- GitHub, accessed on February 24, 2026, [https://github.com/pyusb/pyusb/pull/287](https://github.com/pyusb/pyusb/pull/287)  
12. Access usb devices inside proot \- termux \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/1lyk0vm/access\_usb\_devices\_inside\_proot/](https://www.reddit.com/r/termux/comments/1lyk0vm/access_usb_devices_inside_proot/)  
13. USB/IP Project, accessed on February 24, 2026, [https://usbip.sourceforge.net/](https://usbip.sourceforge.net/)  
14. tools-usb-usbip-README \- The Linux Kernel Archives, accessed on February 24, 2026, [https://www.kernel.org/doc/readme/tools-usb-usbip-README](https://www.kernel.org/doc/readme/tools-usb-usbip-README)  
15. Exploiting USB/IP in Linux UBOAT or CVE-2016-3955 \- Black Hat, accessed on February 24, 2026, [https://blackhat.com/docs/asia-17/materials/asia-17-Korchagin-Exploiting-USBIP-In-Linux-wp.pdf](https://blackhat.com/docs/asia-17/materials/asia-17-Korchagin-Exploiting-USBIP-In-Linux-wp.pdf)  
16. yunsmall/usbipdcpp: A C++ library for creating usbip server. \- GitHub, accessed on February 24, 2026, [https://github.com/yunsmall/usbipdcpp](https://github.com/yunsmall/usbipdcpp)  
17. termux · GitHub Topics, accessed on February 24, 2026, [https://github.com/topics/termux?l=c%2B%2B\&o=desc\&s=updated](https://github.com/topics/termux?l=c%2B%2B&o=desc&s=updated)  
18. bodrick/awesome \- GitHub, accessed on February 24, 2026, [https://github.com/bodrick/awesome](https://github.com/bodrick/awesome)  
19. sovlookup/AWESOME-STARS.md at main \- GitHub, accessed on February 24, 2026, [https://github.com/SOVLOOKUP/sovlookup/blob/main/AWESOME-STARS.md](https://github.com/SOVLOOKUP/sovlookup/blob/main/AWESOME-STARS.md)  
20. dorssel/usbipd-win: Windows software for sharing locally connected USB devices to other machines, including Hyper-V guests and WSL 2\. \- GitHub, accessed on February 24, 2026, [https://github.com/dorssel/usbipd-win](https://github.com/dorssel/usbipd-win)  
21. jaxvanyang/my-awesome-stars \- GitHub, accessed on February 24, 2026, [https://github.com/jaxvanyang/my-awesome-stars](https://github.com/jaxvanyang/my-awesome-stars)  
22. Azathothas/Stars: Automated Cataloguing of Starred Repos because Github Search Sucks, accessed on February 24, 2026, [https://github.com/Azathothas/Stars](https://github.com/Azathothas/Stars)  
23. USB/IP \- ArchWiki, accessed on February 24, 2026, [https://wiki.archlinux.org/title/USB/IP](https://wiki.archlinux.org/title/USB/IP)  
24. Thoughts dereferenced from the scratchpad noise. | Linux, RPi and USB over IP, accessed on February 24, 2026, [https://blog.3mdeb.com/2014/2014-08-18-linux-rpi-and-usb-over-ip/](https://blog.3mdeb.com/2014/2014-08-18-linux-rpi-and-usb-over-ip/)  
25. Bug \#2115827 “vhci-hcd and usbip-core not available” : Bugs : linux-azure package, accessed on February 24, 2026, [https://bugs.launchpad.net/bugs/2115827](https://bugs.launchpad.net/bugs/2115827)  
26. \[Feature Request\] Install kernel module vhci-hcd \- Forums \- Unraid, accessed on February 24, 2026, [https://forums.unraid.net/topic/77325-feature-request-install-kernel-module-vhci-hcd/](https://forums.unraid.net/topic/77325-feature-request-install-kernel-module-vhci-hcd/)  
27. \[PATCH 000/429\] Backport lts patches from Linux-4.19.92 & 93 & 94 \- Kernel \- List Index, accessed on February 24, 2026, [https://mailweb.openeuler.org/archives/list/kernel@openeuler.org/thread/VGPEJFSAQWUPMV3RR6GNQJ6HJGWPIRJD/](https://mailweb.openeuler.org/archives/list/kernel@openeuler.org/thread/VGPEJFSAQWUPMV3RR6GNQJ6HJGWPIRJD/)  
28. How to install the USBIP (vhci\_hcd) kernel module on Jetson Orin NX for 5.10.216-tegra, accessed on February 24, 2026, [https://forums.developer.nvidia.com/t/how-to-install-the-usbip-vhci-hcd-kernel-module-on-jetson-orin-nx-for-5-10-216-tegra/350286](https://forums.developer.nvidia.com/t/how-to-install-the-usbip-vhci-hcd-kernel-module-on-jetson-orin-nx-for-5-10-216-tegra/350286)  
29. 20.04 : scan \- Forum Xubuntu, accessed on February 24, 2026, [https://xubuntu.fr/forum/viewtopic.php?t=4283](https://xubuntu.fr/forum/viewtopic.php?t=4283)  
30. Alia5/VIIPER: Virtual Input over IP Emulator \- VIIPER is a tool to create virtual input devices using USBIP. (Linux/Windows) \- GitHub, accessed on February 24, 2026, [https://github.com/Alia5/VIIPER](https://github.com/Alia5/VIIPER)  
31. viiper-client 0.2.2 on Cargo \- Libraries.io \- security & maintenance, accessed on February 24, 2026, [https://libraries.io/cargo/viiper-client](https://libraries.io/cargo/viiper-client)  
32. Binding devices with libusbip \- usairb devlog \#2, accessed on February 24, 2026, [https://fnune.com/devlog/usairb/2022/03/18/binding-devices-with-libusbip-usairb-devlog-2/](https://fnune.com/devlog/usairb/2022/03/18/binding-devices-with-libusbip-usairb-devlog-2/)  
33. Siemens Third-Party Components in SINEC OS \- CISA, accessed on February 24, 2026, [https://www.cisa.gov/news-events/ics-advisories/icsa-25-226-07](https://www.cisa.gov/news-events/ics-advisories/icsa-25-226-07)  
34. Docker Desktop Mac (Apple Silicon, LinuxKit 6.10.14): kernel oops in usbip/vhci\_hcd when streaming from USB device via VirtualHere (dcache\_inval\_poc → dma\_direct\_unmap\_sg) · Issue \#7791 \- GitHub, accessed on February 24, 2026, [https://github.com/docker/for-mac/issues/7791](https://github.com/docker/for-mac/issues/7791)  
35. Client permanently hangs when tunneling USB sound devices \- VirtualHere, accessed on February 24, 2026, [https://www.virtualhere.com/node/417](https://www.virtualhere.com/node/417)  
36. Installing libusb for nonrooted Android | AnotherTerm-docs, accessed on February 24, 2026, [https://green-green-avk.github.io/AnotherTerm-docs/installing-libusb-for-nonrooted-android.html](https://green-green-avk.github.io/AnotherTerm-docs/installing-libusb-for-nonrooted-android.html)  
37. green-green-avk \- GitHub, accessed on February 24, 2026, [https://github.com/green-green-avk](https://github.com/green-green-avk)  
38. Add termux-api support to libusb \#5831 \- GitHub, accessed on February 24, 2026, [https://github.com/termux/termux-packages/issues/5831](https://github.com/termux/termux-packages/issues/5831)  
39. How to work with usb audio card through X11? : r/termux \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/189y3a3/how\_to\_work\_with\_usb\_audio\_card\_through\_x11/](https://www.reddit.com/r/termux/comments/189y3a3/how_to_work_with_usb_audio_card_through_x11/)  
40. How to access the usb storage (OTG) using the Termux? · Issue \#7037 \- GitHub, accessed on February 24, 2026, [https://github.com/termux/termux-packages/issues/7037](https://github.com/termux/termux-packages/issues/7037)  
41. Use an Android smartphone as a "serial modem" with DOS \-- And "without needing to be root." This "solution works using a QEMU VM running a minimalistic install of NetBSD, which acts as a modem and router for traffic to/from the DOS PC." QEMU, termux-usb, and usbredirect are \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/linux/comments/1hrbyro/use\_an\_android\_smartphone\_as\_a\_serial\_modem\_with/](https://www.reddit.com/r/linux/comments/1hrbyro/use_an_android_smartphone_as_a_serial_modem_with/)  
42. Use an Android smartphone as a "serial modem" with DOS \-- And "without needing to be root." This "solution works using a QEMU VM running a minimalistic install of NetBSD, which acts as a modem and router for traffic to/from the DOS PC." QEMU, termux-usb, and usbredirect are running under Termux. \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/NetBSD/comments/1hrbwaf/use\_an\_android\_smartphone\_as\_a\_serial\_modem\_with/](https://www.reddit.com/r/NetBSD/comments/1hrbwaf/use_an_android_smartphone_as_a_serial_modem_with/)  
43. Attempting to connect QEMU and a USB device with the command 'termux-usb \-e usbredirect device "/dev/bus/usb/001/031" \--to 127.0.0.1:23456' and getting "termux-usb: too many arguments". · Issue \#19635 · termux/termux-packages \- GitHub, accessed on February 24, 2026, [https://github.com/termux/termux-packages/issues/19635](https://github.com/termux/termux-packages/issues/19635)  
44. recipe maintenance \- meta-openembedded \- OpenEmbedded Layer Index, accessed on February 24, 2026, [http://layers.openembedded.org/rrs/recipes/meta-openembedded/4.2/Default/](http://layers.openembedded.org/rrs/recipes/meta-openembedded/4.2/Default/)  
45. Reading and writing a USB drive connected to a Linux server using Termux, termux-usb, usbredirect, and QEMU on a smartphone that is not rooted \- GitHub Gist, accessed on February 24, 2026, [https://gist.github.com/NoteAfterNote/7a197233de3d60ff1e23ca90ed2f595a](https://gist.github.com/NoteAfterNote/7a197233de3d60ff1e23ca90ed2f595a)  
46. Hands-on: We ran full desktop Linux apps on an Android phone\! \-- "With some light setup, you too can run full desktop Linux apps like GIMP and LibreOffice on a Pixel phone" : r/linux \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/linux/comments/1mpwt0n/handson\_we\_ran\_full\_desktop\_linux\_apps\_on\_an/](https://www.reddit.com/r/linux/comments/1mpwt0n/handson_we_ran_full_desktop_linux_apps_on_an/)  
47. Can I dd an USB drive to an image.img file? \- termux \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/11ca2fu/can\_i\_dd\_an\_usb\_drive\_to\_an\_imageimg\_file/](https://www.reddit.com/r/termux/comments/11ca2fu/can_i_dd_an_usb_drive_to_an_imageimg_file/)  
48. USB disk as /dev/sda on a not-rooted smartphone using Termux, QEMU, Alpine Linux, accessed on February 24, 2026, [https://news.ycombinator.com/item?id=40507319](https://news.ycombinator.com/item?id=40507319)  
49. figuring out usb in proot-distro : r/termux \- Reddit, accessed on February 24, 2026, [https://www.reddit.com/r/termux/comments/1bvwfjx/figuring\_out\_usb\_in\_prootdistro/](https://www.reddit.com/r/termux/comments/1bvwfjx/figuring_out_usb_in_prootdistro/)  
50. Forward USB to RPI 4 OTG \- Raspberry Pi Forums, accessed on February 24, 2026, [https://forums.raspberrypi.com/viewtopic.php?t=311026](https://forums.raspberrypi.com/viewtopic.php?t=311026)  
51. Termux (Android support), usb device from sys device (file descriptor), Issue \#285 by Querela · Pull Request \#287 \- GitHub, accessed on February 24, 2026, [https://github.com/pyusb/pyusb/pull/287/files](https://github.com/pyusb/pyusb/pull/287/files)
