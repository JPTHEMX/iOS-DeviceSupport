import UIKit
import SwiftUI
@preconcurrency import Combine

// MARK: - Modelo
struct QuickActionItem: Identifiable {
    let id = UUID()
    var title: String
    var imageName: String
    var url: String
}

enum ViewState {
    case loading
    case error
    case content
}

// MARK: - ViewModel
@MainActor
class QuickActionsViewModel: ObservableObject {
    
    @Published var viewState: ViewState = .loading
    @Published var allOffers: [QuickActionItem] = []
    
    let maxQuickActions = 4
    
    var displayOffers: [QuickActionItem] {
        if allOffers.isEmpty { return [] }
        
        var items = Array(allOffers.prefix(maxQuickActions))
        
        if allOffers.count > maxQuickActions {
            items[maxQuickActions - 1] = QuickActionItem(title: "Más", imageName: "list.bullet", url: "")
        }
        
        return items
    }
    
    func fetchOffers() {
        viewState = .loading
        Task {
            try? await Task.sleep(nanoseconds: 1 * 1_000_000_000)
            
            self.allOffers = [
                QuickActionItem(title: "Test 1", imageName: "", url: "url1"),
                QuickActionItem(title: "Test 2", imageName: "", url: "url2"),
                QuickActionItem(title: "Test 3", imageName: "", url: "url3"),
                QuickActionItem(title: "Test 4", imageName: "", url: "url4"),
                QuickActionItem(title: "Test 5", imageName: "", url: "url5")
            ]
            
            self.viewState = self.allOffers.isEmpty ? .error : .content
            
            try? await Task.sleep(nanoseconds: 2 * 1_000_000_000)
            
            if self.allOffers.count >= 5 {
                self.allOffers[0].imageName = "star.fill"
                self.allOffers[1].imageName = "heart.fill"
                self.allOffers[2].imageName = "bell.fill"
                self.allOffers[3].imageName = "gearshape.fill"
                self.allOffers[4].imageName = "trash.fill"
            }
        }
    }
    
    func selectAction(item: QuickActionItem) {
        if item.imageName == "list.bullet" {
            print("Abrir menú completo")
        } else {
            print("Acción seleccionada: \(item.title) con URL: \(item.url)")
        }
    }
}

// MARK: - QuickActionView
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
                    
                    Image(systemName: item.imageName.isEmpty ? "photo" : item.imageName)
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

// MARK: - QuickActionsView
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
                Text("Error: No hay datos")
            case .loading:
                ProgressView()
                    .frame(maxWidth: .infinity)
            }
        }
        .padding(.zero)
        .frame(maxWidth: .infinity, minHeight: 60)
        .ignoresSafeArea()
        .onAppear {
            viewModel.fetchOffers()
        }
    }
}

// MARK: - QuickActionsViewWrapper
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

// MARK: - ViewController
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
