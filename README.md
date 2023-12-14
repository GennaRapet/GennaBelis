# GennaBelis
//
//  VideoTrimView.swift
//  CollageBuilder
//
//

import SwiftUI
import AVFoundation

struct VideoTrimView: View {
    
    @EnvironmentObject private var store: AppStore
    @Environment(\.dismiss) private var dismiss
    
    @State var trim: VideoTrim?
    
    var url: URL
    var id: String
    
    @State private var isPlaying = false
    @State private var correctSize: CGSize = .zero
    @State private var images: [UIImage] = []
    @State private var correctWidth: CGFloat = .zero
    
    @State private var duration: Double?
    
    var body: some View {
        VStack {
            topBar
            videoView
            .padding()
            .overlay {
                playButton
            }
            scrollView
        }
    }
    
    private var topBar: some View {
        VStack {
            HStack {
                Button {
                    dismiss()
                } label: {
                    Text("Cancel")
                }
                Spacer()
                Button {
                    store.dispatch(.changeCollage(.changeShape(
                        .changeMedia(.changeVideoSettings(.changeTrim(trim))),
                        id: id
                        )))
                    dismiss()
                } label: {
                    Image(systemName: "")
                    Text("Apply")
                }
            }
            .padding(20)
        }
    }
    
    private var playButton: some View {
        Button {
            isPlaying.toggle()
        } label: {
            Image(systemName: isPlaying
                  ? "pause.fill"
                  : "play.fill")
        }
        .font(.largeTitle)
    }
    
    private var videoView: some View {
        GeometryReader { geo in
            VStack {
                VideoPlayerView(
                    videoURL: url,
                    modifiers: [],
                    settings: .init(trim: trim,
                                    speed: 1,
                                    isMuted: true),
                    isPlaying: isPlaying,
                    restartAfterPause: false,
                    context: SharedContext.context
                )
                .frame(width: correctSize.width,
                       height: correctSize.height)
            }
            .frame(maxHeight: .infinity)
            .task {
                await setupCorrectSize(for: geo.size)
            }
        }
    }
    
    private var scrollView: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            let fitWidth = (UIScreen.current?.bounds.width ?? 0) * 2
            HStack(spacing: 0) {
                ForEach(images, id: \.self) {
                    Image(uiImage: $0)
                }
            }
            .overlay {
                if let duration {
                    TrimSelectorView(trim: $trim,
                                     duration: duration,
                                     width: correctWidth)
                }
            }
            .padding(.vertical, 30)
            .padding(.horizontal, 30)
            .task { await setupImages(fitWidth) }
        }
        .frame(height: 200)
    }
    
    private var asset: AVURLAsset {
        AVURLAsset(url: url)
    }
    
    private func setupImages(_ width: CGFloat) async {
        duration = try? await asset.load(
          .duration
        ).seconds
        guard let duration else { return }
        await setupImages(width: width, duration: duration)
    }
    
    private func setupImages(width: CGFloat, duration: Double) async {
        guard let videoSize = await asset.videoSize else {
            return
        }

        let cellHeight: CGFloat = 40
        let imageWidth = videoSize.width / videoSize.height * cellHeight
        let count = Int(width / imageWidth)

        let times = Array(0..<count).map {
            CGFloat($0) / CGFloat(count - 1) * duration
        }
        
        let scale = UIScreen.current?.scale ?? 3
        
        correctWidth = imageWidth * CGFloat(count)
        images = await asset.createImages(for: times).map {
            UIImage(cgImage: $0).resize(
                to: .init(
                    width: imageWidth,
                    height: cellHeight
                ),
                with: scale
            )
        }

    }
    
    private func setupCorrectSize(for geoSize: CGSize) async {
        guard let videoSize = await asset.videoSize else {
            return
        }
        
        let scale = max(videoSize.width / geoSize.width,
                        videoSize.height / geoSize.height
        )
        
        correctSize = .init(width: videoSize.width / scale,
                            height: videoSize.height / scale)
    }
}

struct VideoTrimView_Previews: PreviewProvider {
    static var previews: some View {
        VideoTrimView(url: Bundle.main.url(forResource: "ExampleVideo",
                                           withExtension: "mp4")!,
                      id: "")
        .environmentObject(AppStore.preview)
    }
}

