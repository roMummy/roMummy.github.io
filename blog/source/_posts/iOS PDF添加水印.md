---
title: iOS PDF添加水印
date: 2022-07-20 11:22:37
tags: 学习
categories: iOS
---

## iOS PDF添加水印

最近在做PDF相关的项目，记录一下

### 实现功能

* 支持添加文字水印
* 支持旋转
* 支持导出
* 支持透明度
* 支持图层
* 支持平铺
* 支持添加页面范围
* 支持字体样式设置
* 支持实时变更

### 实现原理

* 通过使用苹果自带的PDFKit来实现。

  PDFDocument有一个delegate，其中有一个代理方法```func classForPage() -> AnyClass```，这个方法可以让我们自己绘制PDFPage。绘制好PDF之后就可以使用PDFDocument提供的```write(toFile path: String) -> Bool```来导出。

###具体实现

* 绘制水印

  创建一个全新的PDFPage

  然后重写```func draw(with box: PDFDisplayBox, to context: CGContext)```方法

  ```swift
  class PDFWatermarkPage: PDFPage {
      var pageIndex: Int? {
          if let label = label {
              return Int(label)
          }
          return nil
      }
      override func draw(with box: PDFDisplayBox, to context: CGContext) {
          super.draw(with: box, to: context)
          let config = PDFWatermarkTool.PageConfiguration.shared.configuration
          
          guard let pageIndex = pageIndex else {
              return
          }
          if pageIndex < config.startPageIndex {
              return
          }
          if pageIndex > config.endPageIndex {
              return
          }
          
          UIGraphicsPushContext(context)
          context.saveGState()
          let pageBounds = bounds(for: box)
          
          print("page - \(label)")
          // 渲染文字
          drawWatermark(config: config, contextWidth: pageBounds.width, contextHeight: pageBounds.height)
          
          context.restoreGState()
          UIGraphicsPopContext()
      }
      /// 计算整个页面可以绘制多少个文本 然后循环绘制文本
      func drawWatermark(config: PDFWatermarkTool.Configuration,
                         contextWidth: CGFloat,
                         contextHeight: CGFloat) {
          guard config.text.count > 0 else {
              return
          }
          // 斜线长
          let sqrtLength = sqrt(contextWidth * contextWidth + contextHeight * contextHeight)
          // 绘制文字的属性
          let attributes = [
              NSAttributedString.Key.foregroundColor: config.textColor.withAlphaComponent(config.textAlpha),
              NSAttributedString.Key.font: config.textFont,
          ]
          let mAttributesStr = NSMutableAttributedString(string: config.text, attributes: attributes)
          // 绘制文字的宽高
          let mStrW = mAttributesStr.size().width
          let mStrH = mAttributesStr.size().height
          // 文字间距
          let verticalSpace: CGFloat = config.textVerticalSpace
          let horizontalSpace: CGFloat = config.textHorizontalSpace
                  
          // 获取上下文
          guard let context = UIGraphicsGetCurrentContext() else {
              return
          }
          
          // 保存上下文，压栈
          context.saveGState()
          // 翻转坐标系，圆点从左下角变化到左上角
          context.translateBy(x: 0, y: contextHeight)
          context.scaleBy(x: 1.0, y: -1.0)
          
          // 移动绘制圆点到画布中心
          context.translateBy(x: contextWidth/2.0, y: contextHeight/2.0)
          // 旋转
          // 导出的时候需要特殊处理旋转角度
          let angle = config.isExport ? (config.textAngle - rotation): config.textAngle
          context.rotate(by: CGFloat(angle)*(CGFloat.pi/180.0))
          // 将绘制原点恢复初始值，保证当前context中心和源image的中心处在一个点(当前context已经旋转，所以绘制出的任何layer都是倾斜的)
          context.translateBy(x: -contextWidth/2.0, y: -contextHeight/2.0)
          
          // layer 
          if config.layer == .bottom {
            	// 正片叠底 详见：https://www.jianshu.com/p/7c92c57cdf6f
              context.setBlendMode(.multiply)
          }
          
          // 计算绘制的行和列
          let rowCount = Int(sqrtLength / (mStrW + horizontalSpace)) + 1
          let columnCount = Int(sqrtLength / (mStrH + verticalSpace)) + 1
          // 此处计算出需要绘制水印文字的起始点，由于水印区域要大于图片区域所以起点在原有基础上移
          let orignX = -(sqrtLength - contextWidth) / 2.0
          let orignY = -(sqrtLength - contextHeight) / 2.0
          
          var tempOrignX = orignX
          var tempOrignY = orignY
          
          for i in 0..<rowCount*columnCount {
              // 渲染文本
              NSString(string: config.text).draw(in: CGRect(x: tempOrignX, y: tempOrignY, width: mStrW, height: mStrH), withAttributes: attributes)
              // 换行
              if i % rowCount == 0 && i != 0 {
                  tempOrignX = orignX
                  tempOrignY += (mStrH + verticalSpace)
              } else {
                  tempOrignX += (mStrW + horizontalSpace)
              }
          }
          context.restoreGState()
      }
  }
  ```

  >特别需要注意的地方有2个点：
  >
  >* CGContext的坐标系(原点在左下角)和UIKit的坐标系(原点在左上角))不一样，需要转换坐标系。
  >* 导出的时候CGContext的坐标系也不一样(原点在右下角)，需要特殊处理```config.textAngle - rotation```水印旋转角度减去页面自身的旋转角度

* 导出水印

  ```swift
  import Foundation
  import UIKit
  import PDFKit
  
  /// PDF页码管理
  class PDFWatermarkTool: NSObject {
      var configuration: Configuration {
          didSet {
              PageConfiguration.shared.configuration = configuration
          }
      }
      let pdfDocument: PDFDocument
      
      init(pdfPath: String, configuration: Configuration = Configuration()) {
          guard let document = PDFDocument(url: URL(fileURLWithPath: pdfPath)) else {
              fatalError("请传入有效的PDF路径")
          }
          self.configuration = configuration
          PageConfiguration.shared.configuration = configuration
          pdfDocument = document
          super.init()
          pdfDocument.delegate = self
      }
      
      func exportFile(complate: @escaping (_ outputPath: String?) -> Void) {
          var newConfiguration = configuration
          newConfiguration.isExport = true
          configuration = newConfiguration
          
          DispatchQueue.global(qos: .background).async {[weak self] in
              guard let `self` = self else {return}
              var path: String?
              path = NSTemporaryDirectory() + "test.pdf"
              if self.pdfDocument.write(toFile: path!) {
                  print("导出文件成功 - \(path!)")
              }
              
              DispatchQueue.main.async {
                  complate(path)
              }
          }
      }
  }
  extension PDFWatermarkTool: PDFDocumentDelegate {
      func classForPage() -> AnyClass {
          return FZPDFWatermarkPage.self
      }
  }
  extension PDFWatermarkTool {
      class PageConfiguration {
          static let shared = PageConfiguration()
          private init() {}
          
          var configuration: FZPDFWatermarkTool.Configuration = FZPDFWatermarkTool.Configuration()
      }
  }
  extension PDFWatermarkTool {
      struct Configuration {
          /// 水印文字
          var text: String = "维权必究"
          /// 文本颜色
          var textColor: UIColor = .red
          /// 文本字体
          var textFont: UIFont = .systemFont(ofSize: 16)
          /// 文字垂直方向间距
          var textVerticalSpace: CGFloat = 50.0
          /// 文字水平方向间距
          var textHorizontalSpace: CGFloat = 50.0
          /// 透明度
          var textAlpha: CGFloat = 1.0
          /// 倾斜角度
          var textAngle: Int = 45
          /// 图层 顶部和底部
          var layer: Layer = .bottom
          
          /// 开始页面位置
          var startPageIndex: Int = 1
          /// 结束页面位置
          var endPageIndex: Int = .max
          /// 是否是导出
          var isExport: Bool = false
      }
      enum Vertical {
          case top
          case bottom
      }
      enum Horizontal {
          case left
          case right
          case center
      }
      enum Layer:Codable, CaseIterable {
          case top
          case bottom
      }
  }
  ```

  TODO: 这里导出的时候会走PDFPage的```func draw(with box: PDFDisplayBox, to context: CGContext)```方法，并且是一页一页的去渲染，最后再写入到文件里面，理论上可以获取导出进度的。

### 页面展示

```swift
import PDFKit
import UIKit

class ViewController: UIViewController {
		private lazy var pdfView: PDFView = {
      	let view = PDFView(frame: self.view.frame)
        view.autoScales = true
        view.isUserInteractionEnabled = true
				view.backgroundColor = .main_bg_blue_color
//        view.displayMode = .twoUp
        view.displayBox = .artBox
//        view.usePageViewController(true, withViewOptions: nil)
        view.displayDirection = .vertical
        view.autoScales = true
    }()

    lazy var pdfWatermark: FZPDFWatermarkTool = {
        let pdfUrl = Bundle.main.url(forResource: "ASample", withExtension: "pdf")
        let config = PDFWatermarkTool.Configuration()

        let pdfWatermark = PDFWatermarkTool(pdfPath: (pdfUrl?.path)!, configuration: config)
        return pdfWatermark
    }()

    override func viewDidLoad() {
        super.viewDidLoad()
        pdfView.document = pdfWatermark.pdfDocument

        view.addSubview(pdfView)
    }
}
```

### 资料

官方添加水印资料: https://developer.apple.com/documentation/pdfkit/custom_graphics

PDFKit苹果官方资料：https://developer.apple.com/documentation/pdfkit

Quartz 2D：https://developer.apple.com/documentation/coregraphics