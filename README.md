//
//  ContentView.swift
//  Metric
//
//  Free starter: records, cancels noise, counts downloads, unlocks Pro.
//

import SwiftUI
import AVFoundation
import StoreKit

struct ContentView: View {
    @StateObject private var recorder = MetricRecorder()
    @StateObject private var store   = MetricStore()
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                Text("Metric").font(.largeTitle.bold())
                
                Button {
                    recorder.toggleRecord()
                } label: {
                    Image(systemName: recorder.isRecording ? "stop.circle" : "record.circle")
                        .font(.system(size: 64))
                        .foregroundColor(recorder.isRecording ? .red : .accentColor)
                }
                
                Text(recorder.statusText)
                    .font(.callout)
                    .foregroundColor(.secondary)
                
                if let url = recorder.lastExport {
                    ShareLink(item: url) {
                        Label("Share", systemImage: "square.and.arrow.up")
                    }
                    .buttonStyle(.borderedProminent)
                }
                
                ProBanner(store: store)
            }
            .padding()
            .navigationDestination(isPresented: $store.showPaywall) {
                PaywallView(store: store)
            }
            .onAppear {
                store.observeTransactionUpdates()
            }
        }
    }
}

// MARK: - Recorder (noise-cancels, headphone-aware)
class MetricRecorder: ObservableObject {
    @Published var isRecording = false
    @Published var statusText   = "Tap to record"
    @Published var lastExport: URL?
    
    private var engine        = AVAudioEngine()
    private var mixerNode     = AVAudioMixerNode()
    private let downloadCounter = DownloadCounter()
    
    init() {
        setupSession()
    }
    
    private func setupSession() {
        let session = AVAudioSession.sharedInstance()
        do {
            try session.setCategory(.playAndRecord,
                                    mode: .default,
                                    options: [.defaultToSpeaker, .allowBluetooth])
            try session.setActive(true)
        } catch { fatalError("Mic \(error)") }
    }
    
    func toggleRecord() {
        isRecording ? stop() : start()
    }
    
    private func start() {
        guard downloadCounter.canDownload() else {
            statusText = "Upgrade to Pro for unlimited exports"
            return
        }
        
        let input = engine.inputNode
        engine.attach(mixerNode)
        engine.connect(input, to: mixerNode, format: nil)
        
        // Voice-isolation ON when no headphones
        if !isHeadsetConnected() {
            enableVoiceIsolation()
        }
        
        let dstURL = FileManager.default.temporaryDirectory
            .appendingPathComponent("metric-\(Date().timeIntervalSince1970).m4a")
        
        let settings = [
            AVFormatIDKey: Int(kAudioFormatMPEG4AAC),
            AVSampleRateKey: 44100,
            AVEncoderBitRateKey: 192_000
        ] as [String : Any]
        
        do {
            let writer = try AVAudioFile(forWriting: dstURL, settings: settings)
            mixerNode.installTap(onBus: 0, bufferSize: 8192, format: nil) { buffer, _ in
                try? writer.write(from: buffer)
            }
            try engine.start()
            isRecording = true
            statusText = "Recording…"
            lastExport = nil
        } catch {
            statusText = "Start failed"
        }
    }
    
    private func stop() {
        mixerNode.removeTap(onBus: 0)
        engine.stop()
        isRecording = false
        statusText = "Saved"
        
        let url = FileManager.default.temporaryDirectory
            .appendingPathComponent("metric-\(Date().timeIntervalSince1970).m4a")
        lastExport = url
        downloadCounter.increment()
    }
    
    private func isHeadsetConnected() -> Bool {
        let route = AVAudioSession.sharedInstance().currentRoute
        return route.outputs.contains { $0.portType == .headsetMic || $0.portType == .bluetoothHFP }
    }
    
    private func enableVoiceIsolation() {
        // iOS 15+ Voice Isolation (on-device AI)
        guard let availableModes = AVAudioSession.sharedInstance().availableModes as? [String] else { return }
        if availableModes.contains(AVAudioSession.Mode.voiceChat.rawValue) {
            do {
                try AVAudioSession.sharedInstance().setMode(.voiceChat)
            } catch { /* fallback */ }
        }
    }
}

// MARK: - Download Counter (free 10)
class DownloadCounter {
    private let key = "metricFreeDownloads"
    private let defaults = UserDefaults.standard
    
    func canDownload() -> Bool {
        let used = defaults.integer(forKey: key)
        return used < 10
    }
    
    func increment() {
        let used = defaults.integer(forKey: key)
        defaults.set(used + 1, forKey: key)
    }
}

// MARK: - StoreKit 2 (1-month trial + 69.99/yr)
class MetricStore: ObservableObject {
    @Published var showPaywall = false
    
    private let productID = "io.metric.daw.pro.yearly" // create in App Store Connect
    private var product: Product?
    
    func observeTransactionUpdates() {
        Task { for await update in Transaction.updates { /* verify receipt */ } }
    }
    
    func purchase() async {
        do {
            let products = try await Product.products(for: [productID])
            guard let pro = products.first else { return }
            _ = try await pro.purchase()
        } catch {
            // handle error
        }
    }
}

struct ProBanner: View {
    @ObservedObject var store: MetricStore
    var body: some View {
        Button("Unlock unlimited exports + AI master") {
            store.showPaywall = true
        }
        .buttonStyle(.bordered)
    }
}

struct PaywallView: View {
    @ObservedObject var store: MetricStore
    var body: some View {
        VStack(spacing: 20) {
            Text("Metric Pro")
                .font(.largeTitle.bold())
            Text("• Unlimited downloads\n• AI mastering\n• Cloud backup")
                .multilineTextAlignment(.center)
            Button {
                Task { await store.purchase() }
            } label: {
                Text("Start 1-month free trial")
                    .frame(maxWidth: .infinity)
            }
            .buttonStyle(.borderedProminent)
            Text("$69.99/year after trial")
                .font(.caption)
                .foregroundColor(.secondary)
        }
        .
- **Workflow permissions** → **Read and write** → **Save**.  
  
