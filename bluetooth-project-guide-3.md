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
    
    init(packetProcessor: BLEPacketProcessor) {
        self.packetProcessor = packetProcessor
        super.init()
        self.centralManager = CBCentralManager(delegate: self, queue: nil)
    }
    
    func startCapture() {
        if centralManager.state == .poweredOn {
            centralManager.scanForPeripherals(withServices: nil, options: [CBCentralManagerScanOptionAllowDuplicatesKey: true])
        }
    }
    
    func stopCapture() {
        centralManager.stopScan()
    }
    
    func centralManager(_ central: CBCentralManager, didDiscover peripheral: CBPeripheral, advertisementData: [String : Any], rssi RSSI: NSNumber) {
        let packet = BLEPacket(peripheral: peripheral, advertisementData: advertisementData, rssi: RSSI)
        packetProcessor.processPacket(packet)
        
        if !discoveredDevices.contains(where: { $0.id == peripheral.identifier }) {
            let device = DiscoveredDevice(id: peripheral.identifier, name: peripheral.name ?? "Unknown", rssi: RSSI.intValue)
            discoveredDevices.append(device)
        }
    }
}

// MARK: - BLE Packet
struct BLEPacket: Identifiable {
    let id = UUID()
    let peripheral: CBPeripheral
    let advertisementData: [String: Any]
    let rssi: NSNumber
    let timestamp = Date()
}

struct DiscoveredDevice: Identifiable {
    let id: UUID
    let name: String
    let rssi: Int
}

// MARK: - BLE Packet Processor
class BLEPacketProcessor {
    private let queue = DispatchQueue(label: "com.example.blepacketprocessor", attributes: .concurrent)
    private let packetSubject = PassthroughSubject<BLEPacket, Never>()
    private var cancellables = Set<AnyCancellable>()
    private let analyzer: BLEPacketAnalyzer
    
    init(analyzer: BLEPacketAnalyzer) {
        self.analyzer = analyzer
        setupPacketProcessing()
    }
    
    private func setupPacketProcessing() {
        packetSubject
            .buffer(size: 100, prefetch: .byRequest, whenFull: .dropOldest)
            .collect(.byTime(DispatchQueue.main, 0.5))
            .sink { [weak self] packets in
                self?.analyzer.processPackets(packets)
            }
            .store(in: &cancellables)
    }
    
    func processPacket(_ packet: BLEPacket) {
        queue.async {
            self.packetSubject.send(packet)
        }
    }
}

// MARK: - BLE Packet Analyzer
class BLEPacketAnalyzer: ObservableObject {
    @Published var packetStatistics: PacketStatistics = PacketStatistics()
    @Published var devicePacketCounts: [UUID: Int] = [:]
    private var allPackets: [BLEPacket] = []
    
    func processPackets(_ packets: [BLEPacket]) {
        allPackets.append(contentsOf: packets)
        updateStatistics(with: packets)
    }
    
    private func updateStatistics(with packets: [BLEPacket]) {
        packetStatistics.totalPackets += packets.count
        
        for packet in packets {
            packetStatistics.rssiValues.append(packet.rssi.intValue)
            devicePacketCounts[packet.peripheral.identifier, default: 0] += 1
            
            if let manufacturerData = packet.advertisementData[CBAdvertisementDataManufacturerDataKey] as? Data {
                packetStatistics.manufacturerData[manufacturerData] = (packetStatistics.manufacturerData[manufacturerData] ?? 0) + 1
            }
        }
        
        packetStatistics.uniqueDevices = Set(packets.map { $0.peripheral.identifier }).count
        packetStatistics.averageRSSI = packetStatistics.rssiValues.reduce(0, +) / packetStatistics.rssiValues.count
    }
    
    func getPacketFlow(for deviceID: UUID) -> [BLEPacket] {
        return allPackets.filter { $0.peripheral.identifier == deviceID }
    }
}

struct PacketStatistics {
    var totalPackets: Int = 0
    var uniqueDevices: Int = 0
    var rssiValues: [Int] = []
    var averageRSSI: Int = 0
    var manufacturerData: [Data: Int] = [:]
}

// MARK: - User Interface
struct ContentView: View {
    @StateObject private var packetAnalyzer = BLEPacketAnalyzer()
    @StateObject private var packetCapture: BLEPacketCapture
    @State private var isCapturing = false
    @State private var selectedDeviceID: UUID?
    
    init() {
        let analyzer = BLEPacketAnalyzer()
        let processor = BLEPacketProcessor(analyzer: analyzer)
        _packetAnalyzer = StateObject(wrappedValue: analyzer)
        _packetCapture = StateObject(wrappedValue: BLEPacketCapture(packetProcessor: processor))
    }
    
    var body: some View {
        NavigationView {
            List(packetCapture.discoveredDevices) { device in
                Text(device.name)
                    .onTapGesture {
                        selectedDeviceID = device.id
                    }
            }
            .listStyle(SidebarListStyle())
            .frame(minWidth: 200)
            
            VStack {
                PacketStatisticsView(statistics: packetAnalyzer.packetStatistics)
                
                if let deviceID = selectedDeviceID {
                    PacketFlowView(packets: packetAnalyzer.getPacketFlow(for: deviceID))
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

struct PacketStatisticsView: View {
    let statistics: PacketStatistics
    
    var body: some View {
        VStack(alignment: .leading) {
            Text("Packet Statistics")
                .font(.headline)
            Text("Total Packets: \(statistics.totalPackets)")
            Text("Unique Devices: \(statistics.uniqueDevices)")
            Text("Average RSSI: \(statistics.averageRSSI) dBm")
            
            Text("Manufacturer Data:")
                .font(.subheadline)
            ForEach(Array(statistics.manufacturerData.keys), id: \.self) { key in
                Text("\(key.hexEncodedString()): \(statistics.manufacturerData[key] ?? 0)")
            }
        }
        .padding()
    }
}

struct PacketFlowView: View {
    let packets: [BLEPacket]
    
    var body: some View {
        VStack {
            Text("Packet Flow")
                .font(.headline)
            
            ScrollView(.horizontal) {
                HStack(spacing: 10) {
                    ForEach(packets) { packet in
                        VStack {
                            Text(packet.timestamp, style: .time)
                            Circle()
                                .fill(Color.blue)
                                .frame(width: 10, height: 10)
                            Text("RSSI: \(packet.rssi.intValue)")
                        }
                    }
                }
            }
        }
        .padding()
    }
}

// MARK: - Helpers
extension Data {
    func hexEncodedString() -> String {
        return map { String(format: "%02hhx", $0) }.joined()
    }
}

// MARK: - App Entry Point
@main
struct BLESnifferApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

This sample project provides a basic implementation of a Bluetooth Low Energy packet sniffer and analyzer. It includes the following main components:

1. `BLEPacketCapture`: Responsible for scanning and capturing BLE packets using Core Bluetooth.
2. `BLEPacketProcessor`: Processes captured packets asynchronously using Combine.
3. `BLEPacketAnalyzer`: Analyzes processed packets and maintains statistics.
4. `ContentView`: The main user interface that displays discovered devices, packet statistics, and packet flow.

To use this project:

1. Create a new SwiftUI project in Xcode.
2. Replace the contents of your main Swift file with the code above.
3. Ensure your app has the necessary permissions to use Bluetooth:
   - Add the "Privacy - Bluetooth Always Usage Description" key to your Info.plist file.
   - Enable the Bluetooth capability in your project's signing & capabilities tab.

This sample project provides a foundation that you can build upon to implement more advanced features, such as:

- Detailed packet parsing for specific BLE protocols
- More sophisticated data visualization (e.g., charts for RSSI over time)
- Advanced security analysis features
- Packet filtering and search capabilities
- Exporting captured data for further analysis

Remember to thoroughly test the application and handle edge cases, such as Bluetooth being turned off or the app not having necessary permissions.

