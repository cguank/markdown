meta-manifest: https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/meta-manifest.sppay.8.json
```json
{"rn_version":"2.1.2","base_url":"https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/","grayscale":10,"snapshot_id":"7d64346c1ca2a9a578e9edb8ca9d1694","min_app_version":10020,"bundle_type":"js","updated_ts":1703574615,"updated_at":"2023-12-26 07:10:74 +0000","grayscale_schedule":[{"time":1703574615,"percentage":10},....{"time":1703579835,"percentage":97},{"time":1703579895,"percentage":98},{"time":1703579955,"percentage":99},{"time":1703580015,"percentage":100}]}
```

main-manifest:- 根据rn_version:2.0.2获取live的main manifest: [https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/manifest.sppay.8.2-0-2.json](https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/manifest.sppay.8.2-0-2.json)
```json
{
  "plugins": [
    {
      "plugin": "base",
      "priority": 0,
      "manifest_url": "https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/b99377103f34a86b9406360f48d6b4a8.manifest.7z",
      "manifest_md5": "b99377103f34a86b9406360f48d6b4a8",
      "zip_url": "https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/498c8ffe3c4366eaa2c8162f809c8bf5.plugin.7z",
      "zip_md5": "498c8ffe3c4366eaa2c8162f809c8bf5",
      "releaseTag": "1.0.4",
      "updated_at": "2023-09-14 04:15:12 +0000"
    },
    ...,
    {
      "plugin": "shopeepay",
      "priority": 1,
      "manifest_url": "https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/95321a209a14645a4914c3077bc3ccdd.manifest.7z",
      "manifest_md5": "95321a209a14645a4914c3077bc3ccdd",
      "zip_url": "https://deo.shopeemobile.com/shopee/shopee-rn-live-id/ios/09933a13b250c8ae440770fb135a4580.plugin.7z",
      "zip_md5": "09933a13b250c8ae440770fb135a4580",
      "releaseTag": "1.62.10-feat-shopeepay-plus.20231207",
      "updated_at": "2023-12-07 09:04:58 +0000"
    },
    ...,
  ]
}
```
### 为什么需要meta-manifest
可以看到服务端比较meta-manifest的版本号来决定是否需要更新，那么比起manifest随着插件增多越来越大，meta-manifest要小很多，能减少CDN带宽。
![[app-fetch-manifest.png]]
### PV 和 RN Bundle Version
- pv来自于部署填的release tag，用来告诉native是否已经加载了bundle
- RN Bundle Version 来自于 host部署前，RM修改的配置json 
![[Pasted image 20240102104931.png]]![[Pasted image 20240102105409.png]]