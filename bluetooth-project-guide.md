[//]: # (Cmd K + V to view)

# Comprehensive Bluetooth Packet Sniffer and Analyzer Project Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Learning Objectives](#learning-objectives)
3. [Project Structure](#project-structure)
4. [Detailed Project Plan](#detailed-project-plan)
   - [Part 1: Bluetooth Basics and Environment Setup](#part-1-bluetooth-basics-and-environment-setup)
   - [Part 2: Packet Capture and Parsing](#part-2-packet-capture-and-parsing)
   - [Part 3: Data Analysis and Visualization](#part-3-data-analysis-and-visualization)
   - [Part 4: Advanced Features and Project Finalization](#part-4-advanced-features-and-project-finalization)
5. [Deliverables](#deliverables)
6. [Evaluation Criteria](#evaluation-criteria)
7. [Appendix: Sample Completed Project](#appendix-sample-completed-project)

## Project Overview

This graduate-level project focuses on developing a Bluetooth packet sniffer and analyzer using macOS. The project is designed to deepen your understanding of Bluetooth technology, protocol analysis, and macOS programming. By the end of this month-long project, you will have created a functional Bluetooth packet sniffer and analyzer that can capture, decode, and analyze Bluetooth Low Energy (BLE) packets.

## Learning Objectives

- Understand Bluetooth Low Energy (BLE) protocol and packet structure
- Gain proficiency in macOS Bluetooth programming using Core Bluetooth framework
- Develop skills in packet capture, parsing, and analysis
- Learn about Bluetooth security and encryption
- Implement data visualization techniques for packet analysis

## Project Structure

The project is divided into four main parts, each building upon the previous ones:

1. Bluetooth Basics and Environment Setup (1 week)
2. Packet Capture and Parsing (1 week)
3. Data Analysis and Visualization (1 week)
4. Advanced Features and Project Finalization (1 week)

## Detailed Project Plan

### Part 1: Bluetooth Basics and Environment Setup (1 week)

#### 1.1 Study Bluetooth Low Energy (BLE) fundamentals

- Research BLE protocol stack
- Understand BLE packet types and structures
- Study BLE advertising and connection processes

**Recommended Reading:**
- "Bluetooth Low Energy: The Developer's Handbook" by Robin Heydon
- Bluetooth SIG's official documentation on BLE: https://www.bluetooth.com/bluetooth-resources/

**Key Concepts to Understand:**
1. BLE Protocol Stack
   - Physical Layer
   - Link Layer
   - Host Controller Interface (HCI)
   - Logical Link Control and Adaptation Protocol (L2CAP)
   - Attribute Protocol (ATT)
   - Generic Attribute Profile (GATT)
   - Security Manager Protocol (SMP)

2. BLE Packet Types
   - Advertising packets
   - Data packets
   - Control packets

3. BLE Advertising Process
   - Advertising channels (37, 38, 39)
   - Advertising packet types (ADV_IND, ADV_DIRECT_IND, ADV_NONCONN_IND, ADV_SCAN_IND)
   - Scanning and connection establishment

**Example: BLE Advertising Packet Structure**

```
+---------------------+------------------------+-----------------------+
| Preamble (1 byte)   | Access Address (4 bytes)| PDU (2-39 bytes)      |
+---------------------+------------------------+-----------------------+
| CRC (3 bytes)       |
+---------------------+

PDU Structure:
+---------------------+------------------------+-----------------------+
| Header (2 bytes)    | MAC Address (6 bytes)  | Advertising Data      |
+---------------------+------------------------+-----------------------+
```

#### 1.2 Set up the development environment on macOS

- Install Xcode and necessary development tools
- Familiarize yourself with the Core Bluetooth framework
- Set up a version control system (e.g., Git) for your project

**Recommended Tools:**
- Xcode (latest version)
- Git for version control
- Homebrew for package management

**Core Bluetooth Framework Overview:**
Core Bluetooth is Apple's framework for communicating with BLE devices. Key classes include:
- `CBCentralManager`: For scanning and connecting to peripherals
- `CBPeripheral`: Represents a remote BLE device
- `CBService`: Represents a service provided by a peripheral
- `CBCharacteristic`: Represents a characteristic of a service

**Example: Setting up a basic Xcode project for BLE**

1. Open Xcode and create a new project (File > New > Project)
2. Choose "App" under macOS
3. Name your project (e.g., "BLEScanner") and choose a location to save it
4. In your project settings, go to the "Signing & Capabilities" tab
5. Add the "Bluetooth" capability

Now, let's set up a basic BLE scanning class:

```swift
import CoreBluetooth

class BLEScanner: NSObject, CBCentralManagerDelegate {
    var centralManager: CBCentralManager!
    
    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }
    
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        switch central.state {
        case .poweredOn:
            print("Bluetooth is powered on")
        case .poweredOff:
            print("Bluetooth is powered off")
        case .unsupported:
            print("Bluetooth is unsupported")
        case .unauthorized:
            print("Bluetooth is unauthorized")
        case .resetting:
            print("Bluetooth is resetting")
        case .unknown:
            print("Bluetooth state is unknown")
        @unknown default:
            print("Unknown Bluetooth state")
        }
    }
}
```

#### 1.3 Create a basic BLE scanner application

- Implement a simple BLE scanner using Core Bluetooth
- Display basic information about discovered BLE devices
- Handle Bluetooth permissions and error scenarios

**Example: Implementing a Basic BLE Scanner**

Let's extend our `BLEScanner` class to perform scanning and display discovered devices:

```swift
class BLEScanner: NSObject, CBCentralManagerDelegate {
    var centralManager: CBCentralManager!
    var discoveredPeripherals: [CBPeripheral] = []
    
    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }
    
    func startScanning() {
        if centralManager.state == .poweredOn {
            centralManager.scanForPeripherals(withServices: nil, options: nil)
            print("Scanning started")
        } else {
            print("Bluetooth is not available")
        }
    }
    
    func stopScanning() {
        centralManager.stopScan()
        print("Scanning stopped")
    }
    
    func centralManager(_ central: CBCentralManager, didDiscover peripheral: CBPeripheral, advertisementData: [String : Any], rssi RSSI: NSNumber) {
        if !discoveredPeripherals.contains(peripheral) {
            discoveredPeripherals.append(peripheral)
            print("Discovered: \(peripheral.name ?? "Unknown Device") - RSSI: \(RSSI)")
        }
    }
    
    // ... (previous methods)
}
```

To use this scanner in your macOS app, you can create an instance of `BLEScanner` in your view controller and call `startScanning()` when appropriate:

```swift
class ViewController: NSViewController {
    let bleScanner = BLEScanner()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        bleScanner.startScanning()
    }
}
```

This basic scanner will print discovered devices to the console. In a real application, you'd want to update the UI with this information.

**Handling Permissions:**
For macOS apps, you need to add a usage description to your `Info.plist` file:

1. Open your project's `Info.plist` file
2. Add a new key: "Privacy - Bluetooth Always Usage Description"
3. Set a value explaining why your app needs Bluetooth access

**Error Handling:**
Always check the `centralManager.state` before attempting to scan or connect. Handle each state appropriately, especially `.poweredOff` and `.unauthorized`.

### Part 2: Packet Capture and Parsing (1 week)

#### 2.1 Implement packet capture functionality

- Extend the scanner to capture raw BLE packets
- Implement packet storage and management
- Ensure efficient handling of continuous packet capture

**Recommended Approach:**
Since Core Bluetooth doesn't provide direct access to raw BLE packets, we'll need to use lower-level APIs. The IOBluetooth framework provides more granular control, but it's a private API and not recommended for App Store distribution. For educational purposes, we'll use it here.

**Example: Capturing Raw BLE Packets**

First, add the IOBluetooth framework to your project:

1. In Xcode, select your project in the navigator
2. Select your target under "TARGETS"
3. Go to the "Build Phases" tab
4. Expand "Link Binary With Libraries"
5. Click the "+" button
6. Search for "IOBluetooth" and add it

Now, let's create a `BLEPacketCapture` class:

```swift
import Foundation
import IOBluetooth

class BLEPacketCapture: NSObject {
    private var capturedPackets: [Data] = []
    
    func startCapture() {
        IOBluetoothHostControllerSupport.enableLinkMonitorMode()
        NotificationCenter.default.addObserver(self, selector: #selector(didReceivePacket(_:)), name: NSNotification.Name("IOBluetoothLinkMonitorPacketNotification"), object: nil)
    }
    
    func stopCapture() {
        IOBluetoothHostControllerSupport.disableLinkMonitorMode()
        NotificationCenter.default.removeObserver(self)
    }
    
    @objc private func didReceivePacket(_ notification: Notification) {
        guard let packet = notification.userInfo?["LinkMonitorPacketData"] as? Data else { return }
        capturedPackets.append(packet)
        print("Captured packet: \(packet.hexEncodedString())")
    }
}

extension Data {
    func hexEncodedString() -> String {
        return map { String(format: "%02hhx", $0) }.joined()
    }
}
```

This class enables the link monitor mode, which allows us to receive raw BLE packets. Each captured packet is stored in the `capturedPackets` array and printed as a hexadecimal string.

#### 2.2 Develop packet parsing and decoding modules

- Create modules to parse different BLE packet types
- Implement decoding for various BLE protocols (e.g., ATT, GATT)
- Develop a system for extensible protocol support

**Example: Basic Packet Parser**

Let's create a simple parser for BLE advertising packets:

```swift
struct BLEPacket {
    let preamble: UInt8
    let accessAddress: UInt32
    let pduType: UInt8
    let pduLength: UInt8
    let advertiserAddress: Data
    let advertisingData: Data
    let crc: UInt32
}

class BLEPacketParser {
    static func parseAdvertisingPacket(_ data: Data) -> BLEPacket? {
        guard data.count >= 10 else { return nil }
        
        let preamble = data[0]
        let accessAddress = data.subdata(in: 1..<5).withUnsafeBytes { $0.load(as: UInt32.self) }
        let pduType = data[5] & 0x0F
        let pduLength = data[5] >> 4
        let advertiserAddress = data.subdata(in: 6..<12)
        let advertisingData = data.subdata(in: 12..<(data.count - 3))
        let crc = data.suffix(3).withUnsafeBytes { $0.load(as: UInt32.self) & 0x00FFFFFF }
        
        return BLEPacket(preamble: preamble,
                         accessAddress: accessAddress,
                         pduType: pduType,
                         pduLength: pduLength,
                         advertiserAddress: advertiserAddress,
                         advertisingData: advertisingData,
                         crc: crc)
    }
}
```

This parser extracts the main components of a BLE advertising packet. You can extend this to handle different PDU types and parse the advertising data further.

#### 2.3 Create a simple command-line interface for the sniffer

- Implement basic commands for starting/stopping packet capture
- Add functionality to display captured packets in real-time
- Implement packet filtering options

**Example: Command-Line Interface**

Here's a basic command-line interface for our BLE sniffer:

```swift
import Foundation

class BLESnifferCLI {
    private let packetCapture = BLEPacketCapture()
    private var isRunning = true
    
    func run() {
        print("BLE Sniffer CLI")
        print("Commands: start, stop, filter, quit")
        
        while isRunning {
            print("> ", terminator: "")
            guard let command = readLine()?.lowercased() else { continue }
            
            switch command {
            case "start":
                packetCapture.startCapture()
                print("Packet capture started")
            case "stop":
                packetCapture.stopCapture()
                print("Packet capture stopped")
            case "filter":
                setFilter()
            case "quit":
                isRunning = false
                packetCapture.stopCapture()
                print("Exiting...")
            default:
                print("Unknown command")
            }
        }
    }
    
    private func setFilter() {
        print("Enter filter (e.g., 'pduType:0' for advertising packets): ")
        if let filter = readLine() {
            // Implement filtering logic here
            print("Filter set: \(filter)")
        }
    }
}

// Usage
let cli = BLESnifferCLI()
cli.run()
```

This CLI allows basic control over the packet capture process. You can extend it to display captured packets in real-time and implement more sophisticated filtering options.

### Part 3: Data Analysis and Visualization (1 week)

#### 3.1 Implement basic packet analysis features

- Develop modules for statistical analysis of captured packets
- Implement packet flow reconstruction
- Create functionality for identifying common BLE protocols and profiles

**Example: Basic Packet Analysis**

Let's create a simple packet analyzer that calculates some basic statistics:

```swift
class BLEPacketAnalyzer {
    private var packets: [BLEPacket] = []
    
    func addPacket(_ packet: BLEPacket) {
        packets.append(packet)
    }
    
    func getStatistics() -> [String: Any] {
        let totalPackets = packets.count
        let uniqueAdvertisers = Set(packets.map { $0.advertiserAddress }).count
        let pduTypeCounts = Dictionary(grouping: packets, by: { $0.pduType })
            .mapValues { $0.count }
        
        return [
            "totalPackets": totalPackets,
            "uniqueAdvertisers": uniqueAdvertisers,
            "pduTypeCounts": pduTypeCounts
        ]
    }
    
    func reconstructFlow(for address: Data) -> [BLEPacket] {
        return packets.filter { $0.advertiserAddress == address }
    }
    
    func identifyProtocols() -> [String: Int] {
        var protocolCounts: [String: Int] = [:]
        
        for packet in packets {
            if packet.advertisingData.