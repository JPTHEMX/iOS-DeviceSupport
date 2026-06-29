import UIKit
import SwiftUI
@preconcurrency import Combine

struct QuickActionItem: Identifiable {
    let id = UUID()
    var title: String
    var imageName: String
    var image: Image?
    var url: String
}

enum ViewState {
    case loading
    case error
    case content
}

@MainActor
class QuickActionsViewModel: ObservableObject {
    
    @Published var viewState: ViewState = .loading
    @Published var allOffers: [QuickActionItem] = []
    
    let maxQuickActions = 4
    
    var displayOffers: [QuickActionItem] {
        if allOffers.isEmpty { return [] }
        var items = Array(allOffers.prefix(maxQuickActions))
        if allOffers.count > maxQuickActions {
            items[maxQuickActions - 1] = QuickActionItem(title: "More", imageName: "list.bullet", image: Image(systemName: "list.bullet"), url: "")
        }
        return items
    }
    
    // Función principal asíncrona
    func fetchOffers() async {
        viewState = .loading
        
        // Simulación de solicitud de ofertas
        let offers = await requestOffers()
        
        if offers.isEmpty {
            viewState = .error
            return
        }
        
        self.allOffers = offers
        self.viewState = .content
        
        // Inicio de descarga de imágenes en paralelo
        await downloadAllImages()
    }
    
    private func requestOffers() async -> [QuickActionItem] {
        try? await Task.sleep(nanoseconds: 1 * 1_000_000_000)
        return [
            QuickActionItem(title: "Test 1", imageName: "", image: nil, url: "url1"),
            QuickActionItem(title: "Test 2", imageName: "", image: nil, url: "url2"),
            QuickActionItem(title: "Test 3", imageName: "", image: nil, url: "url3"),
            QuickActionItem(title: "Test 4", imageName: "", image: nil, url: "url4"),
            QuickActionItem(title: "Test 5", imageName: "", image: nil, url: "url5")
        ]
    }
    
    private func downloadAllImages() async {
        // Simulación de múltiples peticiones de imágenes
        for i in 0..<self.allOffers.count {
            // Aquí llamarías a tu servicio de descarga real
            try? await Task.sleep(nanoseconds: 500_000_000)
            
            let systemNames = ["star.fill", "heart.fill", "bell.fill", "gearshape.fill", "trash.fill"]
            if i < systemNames.count {
                // Al modificar la propiedad @Published, @MainActor garantiza la actualización segura
                self.allOffers[i].image = Image(systemName: systemNames[i])
            }
        }
    }
    
    func selectAction(item: QuickActionItem) {
        print("Selected action: \(item.title)")
    }
}

struct QuickActionView: View {
    var item: QuickActionItem
    var onSelect: () -> Void
    
    var body: some View {
        Button(action: onSelect) {
            VStack(spacing: 4) {
                ZStack {
                    RoundedRectangle(cornerRadius: 12)
                        .fill(Color.gray)
                        .frame(width: 60, height: 60)
                    
                    (item.image ?? Image(systemName: "photo"))
                        .resizable()
                        .scaledToFit()
                        .frame(width: 33.68, height: 33.68)
                }
                Text(item.title)
                    .font(.system(size: 12, weight: .medium))
                    .multilineTextAlignment(.center)
                    .lineLimit(3)
                    .lineSpacing(4)
                    .fixedSize(horizontal: false, vertical: true)
                    .frame(maxWidth: .infinity)
            }
            .frame(maxWidth: 72, alignment: .top)
        }
        .buttonStyle(PlainButtonStyle())
    }
}

struct QuickActionsView: View {
    @ObservedObject var viewModel: QuickActionsViewModel
    
    var body: some View {
        HStack(alignment: .top, spacing: 0) {
            switch viewModel.viewState {
            case .content:
                ForEach(viewModel.displayOffers) { item in
                    QuickActionView(item: item) {
                        viewModel.selectAction(item: item)
                    }
                    if item.id != viewModel.displayOffers.last?.id {
                        Spacer()
                    }
                }
            case .error:
                Text("Error: No data available")
            case .loading:
                ProgressView()
                    .frame(maxWidth: .infinity)
            }
        }
        .padding(.zero)
        .frame(maxWidth: .infinity, minHeight: 60)
        .ignoresSafeArea()
        .task {
            await viewModel.fetchOffers()
        }
    }
}

class QuickActionsViewWrapper: UIView {
    var viewModel: QuickActionsViewModel?
    var contectView: QuickActionsView?
    var hostingController: UIViewController?
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupSwiftUIView()
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        setupSwiftUIView()
    }
    
    override func layoutSubviews() {
        super.layoutSubviews()
        hostingController?.view.frame = self.bounds
    }
    
    private func setupSwiftUIView() {
        viewModel = .init()
        guard let viewModel else { return }
        contectView = QuickActionsView(viewModel: viewModel)
        hostingController = addSwiftUIView(view: contectView!)
    }
    
    func addSwiftUIView<T: View>(view: T) -> UIHostingController<T> {
        let hostingController = UIHostingController(rootView: view)
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        hostingController.view.backgroundColor = .clear
        addSubview(hostingController.view)
        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: topAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: bottomAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: trailingAnchor)
        ])
        return hostingController
    }
}

class ViewController: UIViewController {
    private let mainStackView: UIStackView = {
        let stackView = UIStackView()
        stackView.axis = .vertical
        stackView.spacing = 20
        stackView.translatesAutoresizingMaskIntoConstraints = false
        return stackView
    }()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .white
        setupUI()
    }
    
    private func setupUI() {
        view.addSubview(mainStackView)
        let quickActionsWrapper = QuickActionsViewWrapper()
        mainStackView.addArrangedSubview(quickActionsWrapper)
        NSLayoutConstraint.activate([
            mainStackView.centerYAnchor.constraint(equalTo: view.centerYAnchor),
            mainStackView.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
            mainStackView.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16)
        ])
    }
}
