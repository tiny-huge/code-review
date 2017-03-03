# 移动在线缓存

## 一.在线缓存的必要性
   众所周知，移动一张图目前包含两个应用场景：在线和离线。在线场景中，多用户共用数据，与云服务同步，实时性较强。而后者为单用户使用数据，无实时性要求，不常更新，需要数据拷贝，数据也能保证一定的隐秘性。那么在线缓存的应用场景突出在哪里呢？
###1.在线离线一体化
目前移动一张图的在线离线需要在登录时用户手动切换。而数据方面二者并不是同步的，在数据管理方面存在一定的弊端。而通过在线缓存即可实现多用户同步数据，具有较强的数据实时性,数据也便于云端管理。
###2. 方便用户操作。

用户在在线与离线的应用场景中可以自由切换,比如在线过程中，临时切换，并不需要返回登录页面切换。

但同时，部分用户需要保证数据安全性（不需要在线版）,或者大部分使用场景都是离线,所以我们并不能将离线场景去除。

综述,在线、离线、在线缓存为3大应用场景。

## 二.在线缓存的可行性
目前我们一张图主要数据有两类：地图数据和业务数据。而 ArcGIS for desktop 10.2.2以上已经支持两者缓存的技术可行性。具体条件如下：

* ArcGIS for desktop 版本为10.2.2或者 以上。
* 相关服务需要配置好设置(具体见后文)
* AGS-Runtime-SDK-iOS 10.2.5 
* 业务数据需要通过Feature Service服务.

## 三. 支持在线缓存的服务设置
* 设置允许客户端导出切片
* 设置允许切片导出限制
* 发布[ *Feature Service* ](http://www.cnblogs.com/gis-luq/p/5857188.html)服务

## 四. iOS相关API与代码
### 相关类与方法
* `AGSExportTileCacheTask`
* `AGSExportTileCacheParams`
* `AGSGDBSyncTask `
* `AGSGDBGenerateParameters `
* `exportTileCacheWithParameters`
* `estimateTileCacheSizeWithParameters`
* `generateGeodatabaseWithParameters`

### 初始化底图与要素
	let huaianServiceUrl = "http://192.168.1.162:6080/arcgis/rest/services/test3/MapServer"
	let huaianFeatureUrl = "http://192.168.1.162:6080/arcgis/rest/services/test4/FeatureServer"

	class ViewController: UIViewController,AGSLayerDelegate {

    @IBOutlet var mapView: AGSMapView!
    
    var mapLayer: AGSTiledMapServiceLayer!
    
    var tileCacheTask: AGSExportTileCacheTask!
    
    var gdbTask: AGSGDBSyncTask!
    
    
    
    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
      
        self.mapLayer = AGSTiledMapServiceLayer(URL: NSURL(string: huaianServiceUrl))
        self.mapLayer.delegate = self
        mapView.addMapLayer(mapLayer)
        
        if self.tileCacheTask == nil {
            self.tileCacheTask = AGSExportTileCacheTask(URL: NSURL(string: huaianServiceUrl))
        }
        
    }
	func layerDidLoad(layer: AGSLayer!) {
        if layer == self.mapLayer {
        gdbTask = AGSGDBSyncTask(URL: NSURL(string: huaianFeatureUrl))
        gdbTask.loadCompletion = {
            error in
            if error != nil {
                print(error.localizedDescription)
            }
            for info in self.gdbTask.featureServiceInfo.layerInfos as! [AGSMapServiceLayerInfo] {
                let url = self.gdbTask.URL.URLByAppendingPathComponent("\(info.layerId)")
                let featureServiceTable = AGSGDBFeatureServiceTable(serviceURL: url, credential: self.gdbTask.credential, spatialReference: self.mapView.spatialReference)
                let featureLayer = AGSFeatureTableLayer(featureTable:featureServiceTable)
                self.mapView.addMapLayer(featureLayer)
                
            }
        }
        }

    }
 
### 下载底图形成tpk文件
	@IBAction func downloadTpkAction(sender: UIBarButtonItem) {
        
        
        SVProgressHUD.showWithStatus("准备下载")
        let levels = [self.mapLayer.currentLOD().level]
        let extent = self.mapView.visibleAreaEnvelope
        
        //let extent = AGSEnvelope(xmin: 610875.41170353664, ymin: 3780046.638612428, xmax: 610948.9641964633, ymax: 3780074.6404850003, spatialReference: self.mapView.spatialReference)
        
        let params = AGSExportTileCacheParams(levelsOfDetail: levels, areaOfInterest: extent)
        
        
        //showSizesOfTpk(params)
        downloadTileCache(params)
        
    }

	func downloadTileCache(params: AGSExportTileCacheParams) {
        
        let tpkPath = NSHomeDirectory() + "/Documents" + "/tpk"
        
        self.tileCacheTask.exportTileCacheWithParameters(params, downloadFolderPath: tpkPath, useExisting: true, status: { (status, userInfo) in
            print("\(AGSResumableTaskJobStatusAsString(status)), \(userInfo)")
            let totalBytesDownloaded: AnyObject? = userInfo?["AGSDownloadProgressTotalBytesDownloaded"]
            let totalBytesExpected: AnyObject? = userInfo?["AGSDownloadProgressTotalBytesExpected"]
            if totalBytesDownloaded != nil && totalBytesExpected != nil {
                let dPercentage = Float(totalBytesDownloaded! as! NSNumber)/Float(totalBytesExpected! as! NSNumber)
                SVProgressHUD.showProgress(dPercentage, status: "下载中")
                if dPercentage == 1 {
                    SVProgressHUD.dismiss()
                }
                print(dPercentage)
        
            } }) { (tiledLayer, error) in
                
                if error != nil {
                    SVProgressHUD.showErrorWithStatus(error.localizedDescription)
                }
                self.mapView.reset()
                self.mapView.addMapLayer(tiledLayer, withName:"offline")

        }
    }

###下载要素形成 gdb
	@IBAction func downloadGdbAction(sender: UIBarButtonItem) {
    
        SVProgressHUD.showWithStatus("准备下载")

        let gdbPath = NSHomeDirectory() + "/Documents" + "/gdb"
        let params = AGSGDBGenerateParameters(featureServiceInfo: self.gdbTask.featureServiceInfo)
       
        params.extent = self.mapView.maxEnvelope

        params.outSpatialReference = self.mapView.spatialReference
        let layers = self.gdbTask.featureServiceInfo.layerInfos.map { layerInfo -> Int in layerInfo.layerId
        }
        params.layerIDs = layers
        
        self.gdbTask.generateGeodatabaseWithParameters(params, downloadFolderPath: gdbPath, useExisting: true, status: {
            (status, userInfo) in
                print(AGSResumableTaskJobStatusAsString(status))
            let totalBytesDownloaded: AnyObject? = userInfo?["AGSDownloadProgressTotalBytesDownloaded"]
            let totalBytesExpected: AnyObject? = userInfo?["AGSDownloadProgressTotalBytesExpected"]
            if totalBytesDownloaded != nil && totalBytesExpected != nil {
                let dPercentage = Float(totalBytesDownloaded! as! NSNumber)/Float(totalBytesExpected! as! NSNumber)
                SVProgressHUD.showProgress(dPercentage, status: "下载中")
                if dPercentage == 1 {
                    SVProgressHUD.dismiss()
                }
                print(dPercentage)
            }
            }) { (gdb, error) in
                if error != nil {
                    print(error)
                }
                for layer in self.mapView.mapLayers as! [AGSLayer] {
                    if layer is AGSFeatureTableLayer {
                        self.mapView.removeMapLayer(layer)
                    }
                }
                if gdb.featureTables().count > 0 {
                    for table in gdb.featureTables() as! [AGSFeatureTable]{
                        if table.hasGeometry() {
                        let tableLayer = AGSFeatureTableLayer(featureTable: table)
                        self.mapView.addMapLayer(tableLayer)
                        }
               }           }
        }   
    }

五. 总结与后续问题思考
	
经过研究实践，移动在线缓存技术是完全可行的,详见 CacheDemo