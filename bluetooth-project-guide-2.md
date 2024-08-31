### Part 3: Data Analysis and Visualization (1 week)

#### 3.1 Implement basic packet analysis features (continued)

Let's continue with our `BLEPacketAnalyzer` class, adding more advanced features:

```swift
class BLEPacketAnalyzer {
    // ... (previous code)

    func identifyProtocols() -> [String: Int] {
        var protocolCounts: [String: Int] = [:]
        
        for packet in packets {
            if packet.advertisingData.contains([0x03, 0x03]) {
                protocolCounts["Apple iBeacon", default: 0] += 1
            } else if packet.advertisingData.contains([0x06, 0x16]) {
                protocolCounts["Eddystone", default: 0] += 1
            }
            // Add more protocol identifications here
        }
        
        return protocolCounts
    }
    
    func analyzeRSSI() -> [String: Double] {
        let rssiValues = packets.compactMap { Int(exactly: $0.rssi) }
        guard !rssiValues.isEmpty else { return [:] }
        
        let average = Double(rssiValues.reduce(0, +)) / Double(rssiValues.count)
        let sorted = rssiValues.sorted()
        let median = rssiValues.count % 2 == 0 ?
            Double(sorted[rssiValues.count/2 - 1] + sorted[rssiValues.count/2]) / 2.0 :
            Double(sorted[rssiValues.count/2])
        
        return [
            "average": average,
            "median": median,
            "min": Double(sorted.first!),
            "max": Double(sorted.last!)
        ]
    }
}
```

This extended `BLEPacketAnalyzer` class now includes methods for identifying common BLE protocols (like iBeacon and Eddystone) and analyzing RSSI (Received Signal Strength Indicator) values.

#### 3.2 Develop data visualization components

- Implement real-time graphs for packet statistics
- Create visual representations of packet flows
- Develop a timeline view for captured packets

For data visualization in a macOS app, we'll use SwiftUI, which provides a declarative way to create user interfaces. First, make sure your project is set up to use SwiftUI.

**Example: Packet Statistics Chart**

```swift
import SwiftUI
import Charts

struct PacketStatisticsView: View {
    @ObservedObject var analyzer: BLEPacketAnalyzer
    
    var body: some View {
        VStack {
            Text("Packet Type Distribution")
                .font(.headline)
            
            Chart {
                ForEach(analyzer.getStatistics().pduTypeCounts.sorted(by: { $0.key < $1.key }), id: \.key) { key, value in
                    BarMark(
                        x: .value("PDU Type", String(key)),
                        y: .value("Count", value)
                    )
                }
            }
            .frame(height: 300)
            
            Text("Total Packets: \(analyzer.getStatistics().totalPackets)")
            Text("Unique Advertisers: \(analyzer.getStatistics().uniqueAdvertisers)")
        }
        .padding()
    }
}
```

This SwiftUI view creates a bar chart showing the distribution of packet types, along with some summary statistics.

**Example: Packet Flow Timeline**

```swift
struct PacketFlowTimelineView: View {
    @ObservedObject var analyzer: BLEPacketAnalyzer
    let address: Data
    
    var body: some View {
        ScrollView(.horizontal) {
            LazyHStack(spacing: 10) {
                ForEach(analyzer.reconstructFlow(for: address), id: \.timestamp) { packet in
                    VStack {
                        Text(packet.timestamp.formatted(.dateTime.hour().minute().second()))
                        Circle()
                            .fill(color(for: packet.pduType))
                            .frame(width: 10, height: 10)
                        Text("Type: \(packet.pduType)")
                    }
                }
            }
        }
        .frame(height: 100)
    }
    
    func color(for pduType: UInt8) -> Color {
        switch pduType {
        case 0: return .blue  // ADV_IND
        case 1: return .green // ADV_DIRECT_IND
        case 2: return .orange // ADV_NONCONN_IND
        case 3: return .red // SCAN_REQ
        case 4: return .purple // SCAN_RSP
        default: return .gray
        }
    }
}
```

This view creates a horizontal timeline of packets from a specific advertiser, color-coded by PDU type.

#### 3.3 Create a graphical user interface (GUI) for the application

- Design and implement a user-friendly GUI using AppKit or SwiftUI
- Integrate the CLI functionality into the GUI
- Implement data export and import features

Let's create a main view that combines our visualization components and controls:

```swift
import SwiftUI

struct ContentView: View {
    @StateObject var packetCapture = BLEPacketCapture()
    @StateObject var analyzer = BLEPacketAnalyzer()
    @State private var isCapturing = false
    @State private var selectedAddress: Data?
    
    var body: some View {
        NavigationView {
            List(packetCapture.discoveredDevices, id: \.address) { device in
                Text(device.name ?? "Unknown Device")
                    .onTapGesture {
                        selectedAddress = device.address
                    }
            }
            .listStyle(SidebarListStyle())
            .frame(minWidth: 200)
            
            VStack {
                PacketStatisticsView(analyzer: analyzer)
                
                if let address = selectedAddress {
                    PacketFlowTimelineView(analyzer: analyzer, address: address)
                }
                
                HStack {
                    Button(isCapturing ? "Stop Capture" : "Start Capture") {
                        isCapturing.toggle()
                        if isCapturing {
                            packetCapture.startCapture()
                        } else {
                            packetCapture.stopCapture()
                        }
                    }
                    
                    Button("Export Data") {
                        exportData()
                    }
                }
                .padding()
            }
        }
        .frame(minWidth: 600, minHeight: 400)
    }
    
    func exportData() {
        // Implement data export functionality
    }
}
```

This `ContentView` combines all our components into a cohesive user interface. It includes a sidebar with discovered devices, packet statistics, a packet flow timeline for the selected device, and controls for starting/stopping capture and exporting data.

### Part 4: Advanced Features and Project Finalization (1 week)

#### 4.1 Implement advanced analysis features

- Add support for analyzing encrypted BLE packets
- Implement anomaly detection in packet streams
- Develop pattern recognition for common BLE interactions

**Example: Encrypted Packet Analysis**

Analyzing encrypted BLE packets is complex and typically requires the encryption key. However, we can detect encrypted packets and provide some basic information:

```swift
extension BLEPacketAnalyzer {
    func analyzeEncryptedPackets() -> [String: Any] {
        let encryptedPackets = packets.filter { $0.isEncrypted }
        let totalEncrypted = encryptedPackets.count
        let percentageEncrypted = Double(totalEncrypted) / Double(packets.count) * 100
        
        return [
            "totalEncryptedPackets": totalEncrypted,
            "percentageEncrypted": percentageEncrypted
        ]
    }
}

extension BLEPacket {
    var isEncrypted: Bool {
        // Check if the packet is a data PDU (LL Data PDU)
        guard pduType == 0x02 else { return false }
        
        // The LLID (LSB of the first octet of the payload) indicates if it's encrypted
        // 0b01: Unencrypted
        // 0b10 or 0b11: Encrypted
        let llid = payload.first! & 0x03
        return llid == 0x02 || llid == 0x03
    }
}
```

**Example: Anomaly Detection**

Here's a simple anomaly detection method based on packet frequency:

```swift
extension BLEPacketAnalyzer {
    func detectAnomalies(timeWindow: TimeInterval = 1.0) -> [Anomaly] {
        var anomalies: [Anomaly] = []
        let groupedPackets = Dictionary(grouping: packets) { $0.advertiserAddress }
        
        for (address, devicePackets) in groupedPackets {
            let packetCounts = devicePackets.reduce(into: [:] as [TimeInterval: Int]) { counts, packet in
                let timeKey = floor(packet.timestamp.timeIntervalSince1970 / timeWindow) * timeWindow
                counts[timeKey, default: 0] += 1
            }
            
            let avgCount = Double(devicePackets.count) / Double(packetCounts.count)
            let stdDev = sqrt(packetCounts.values.map { pow(Double($0) - avgCount, 2) }.reduce(0, +) / Double(packetCounts.count))
            
            for (time, count) in packetCounts {
                if abs(Double(count) - avgCount) > 3 * stdDev {
                    anomalies.append(Anomaly(address: address, time: time, count: count, average: avgCount))
                }
            }
        }
        
        return anomalies
    }
}

struct Anomaly {
    let address: Data
    let time: TimeInterval
    let count: Int
    let average: Double
}
```

This method detects anomalies by identifying time windows where the packet count deviates significantly from the average.

#### 4.2 Add Bluetooth security analysis capabilities

- Implement checks for common BLE security vulnerabilities
- Add functionality to detect potential Man-in-the-Middle attacks
- Develop a module for analyzing BLE pairing processes

**Example: Security Analysis Module**

```swift
class BLESecurityAnalyzer {
    func analyzeSecurityFeatures(packets: [BLEPacket]) -> [String: Any] {
        var results: [String: Any] = [:]
        
        // Check if LE Secure Connections is used
        results["usesLESecureConnections"] = packets.contains { $0.advertisingData.contains([0x11, 0x07]) }
        
        // Check for potential MITM vulnerability
        results["potentialMITMVulnerability"] = !packets.contains { $0.advertisingData.contains([0x08, 0x07]) }
        
        // Analyze pairing process
        let pairingPackets = packets.filter { $0.isPairingPacket }
        results["pairingMethodUsed"] = analyzePairingMethod(pairingPackets)
        
        return results
    }
    
    private func analyzePairingMethod(_ packets: [BLEPacket]) -> String {
        // Simplified pairing method analysis
        if packets.contains(where: { $0.payload.contains([0x04]) }) {
            return "Just Works"
        } else if packets.contains(where: { $0.payload.contains([0x05]) }) {
            return "Passkey Entry"
        } else if packets.contains(where: { $0.payload.contains([0x06]) }) {
            return "Out of Band"
        } else {
            return "Unknown"
        }
    }
}

extension BLEPacket {
    var isPairingPacket: Bool {
        // Check if it's a Security Manager Protocol packet
        guard pduType == 0x04 else { return false }
        
        // Check for pairing request/response opcodes
        return payload.first == 0x01 || payload.first == 0x02
    }
}
```

This `BLESecurityAnalyzer` class provides basic security analysis, including checks for LE Secure Connections, potential MITM vulnerabilities, and pairing method analysis.

#### 4.3 Optimize performance and refine the user interface

- Optimize packet capture and processing for better performance
- Implement multi-threading for improved responsiveness
- Refine the GUI based on usability testing

**Example: Multi-threaded Packet Processing**

```swift
import Foundation
import Combine

class BLEPacketProcessor {
    private let queue = DispatchQueue(label: "com.example.blepacketprocessor", attributes: .concurrent)
    private let packetSubject = PassthroughSubject<BLEPacket, Never>()
    private var cancellables = Set<AnyCancellable>()
    
    init(analyzer: BLEPacketAnalyzer) {
        packetSubject
            .buffer(size: 100, prefetch: .byRequest, whenFull: .dropOldest)
            .collect(.byTime(DispatchQueue.main, 0.5))
            .sink { [weak analyzer] packets in
                analyzer?.processPackets(packets)
            }
            .store(in: &cancellables)
    }
    
    func processPacket(_ packet: BLEPacket) {
        queue.async {
            // Perform any necessary pre-processing
            self.packetSubject.send(packet)
        }
    }
}
```

This `BLEPacketProcessor` class uses a concurrent dispatch queue and Combine to process packets in batches, improving performance and responsiveness.

#### 4.4 Prepare project documentation and presentation

- Write comprehensive documentation for the project
- Prepare a technical report detailing the implementation and findings
- Create a presentation showcasing the project's features and results

**Example: Project Documentation Outline**

1. Introduction
   - Project overview
   - Objectives
   - Technologies used

2. System Architecture
   - High-level design
   - Component interactions
   - Data flow

3. Implementation Details
   - Packet capture mechanism
   - Parsing and analysis algorithms
   - User interface design
   - Security analysis features

4. Performance Optimizations
   - Multi-threading approach
   - Data structure optimizations
   - GUI responsiveness improvements

5. Findings and Results
   - Common BLE usage patterns observed
   - Security vulnerabilities detected
   - Performance metrics

6. Challenges and Solutions
   - Dealing with encrypted packets
   - Handling high-volume data streams
   - Ensuring cross-platform compatibility

7. Future Improvements
   - Additional protocol support
   - Machine learning for anomaly detection
   - Integration with other security tools

8. Conclusion
   - Project outcomes
   - Lessons learned
   - Potential real-world applications

## Deliverables

1. Source code for the Bluetooth packet sniffer and analyzer
2. Technical report documenting the project implementation, challenges, and results
3. User manual for the developed application
4. Presentation slides summarizing the project

## Evaluation Criteria

- Functionality and reliability of the packet sniffer and analyzer
- Depth of understanding demonstrated in the implementation of BLE protocols
- Quality and clarity of code and documentation
- Effectiveness of data analysis and visualization features
- Creativity in addressing challenges and implementing advanced features
- Overall presentation and usability of the final application

## Appendix: Sample Completed Project

Here's a simplified version of the main components of the completed project:

```swift
import SwiftUI
import CoreBluetooth
import Combine

// MARK: - BLE Packet Capture
class BLEPacketCapture: NSObject, ObservableObject, CBCentralManagerDelegate {
    @Published var discoveredDevices: [DiscoveredDevice] = []
    private var centralManager: CBCentralManager!
    private let packetProcessor: BLEPacketProcessor
    
    init(