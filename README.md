# LSAppUpgrade
App升级方案
##企业版
企业版的升级方案比较简单，直接拿本地的版本与服务器的版本进行比较，如果服务器版本号大于本地版本，就提示用户升级，版本号一样则不提示。
具体实现代码如下：
```objc
#pragma mark - 检测版本更新
-(void)checkVersion{
    
    NSMutableDictionary *params = [[NSMutableDictionary alloc] init];
    [params setValue:@"KCKP_Leader" forKey:@"arg1"];
    
    [[Globle getInstance].service requestWithServiceIP:UpdateURL ServiceName:@"lbcp_getAppVersion" params:params httpMethod:@"POST" resultIsDictionary:YES completeBlock:^(id result) {
        
        if (nil != result) {
            NSString *jsonStr = [Util objectToJson:result];
            NSLog(@"版本检测%@",jsonStr);
            verModel = [[AppVerModel alloc]initWithString:jsonStr error:nil];

            remark = verModel.remarks;

            localVersion = VersionCode;
            NSLog(@"当前版本号%@",verModel.appversion);
            int localVersionNUm = (localVersion == nil ? -1 : [localVersion intValue]);
            
            //获取服务器版本
            serverVersion = verModel.appversion;
            int serverVersionNum = (serverVersion == nil ? -1 : [serverVersion intValue]);
            //判断是否升级
            if(localVersionNUm < serverVersionNum)
            {
                NSString *upgrade = verModel.upgrade;
                if([@"1" isEqualToString:upgrade])    //   强制升级
                {
                    self.alertView = [[UIAlertView alloc] initWithTitle:@"温馨提示" message:@"有新的版本，请及时更新。" delegate:self cancelButtonTitle:nil otherButtonTitles:@"确定", nil];
                }
                else     //  自选升级
                {
                    self.alertView = [[UIAlertView alloc] initWithTitle:@"温馨提示" message:@"有新的版本，请及时更新。" delegate:self cancelButtonTitle: nil otherButtonTitles:@"确定",@"取消",nil];
                }
                [self.alertView show];

            }else{
            
            }
        }

    }];
    
}

#pragma mark - UIAlertView delegate
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex{
    if(alertView == self.alertView)
    {
        if(buttonIndex == 0)
        {
            [[NSUserDefaults standardUserDefaults] synchronize];
            [[UIApplication sharedApplication] openURL:[NSURL URLWithString:remark]];
        }
    }
}
```

##AppStore版
和企业版的升级原理一样，只不过步骤稍微复杂一点，然后用根据AppID拼接成的链接去苹果服务器获取商店版本，把商店版本的本地版本进行比对，不一致则提示升级，一致则不进行任何操作。
关于版本号大小比较：二位版本号可以直接用浮点数进行比较，三位的话可以用字符串操作过滤小数点后再进行比较。

```objc
//首先要从info.plist文件中获取到本地版本
NSDictionary *infoDic = [[NSBundle mainBundle] infoDictionary];
localVersion = [infoDic objectForKey:@"CFBundleShortVersionString"];

    //获取AppStore的版本
    [manager GET:urlString parameters:nil success:^(AFHTTPRequestOperation * operation, id responseObject) {
        
        if (nil != operation.responseString) {
            
            NSLog(@"版本数据 -> %@",operation.responseString);
            
            verModel = [[VerModel alloc]initWithString:operation.responseString error:nil];
//            NSLog(@"Count -> %zi",verModel.results.count);
            
            NSString *storeVersion;
            if (verModel.results.count > 0) {
                VerResultModel *resModel = verModel.results[0];
                appStoreUrl = resModel.trackViewUrl;
                storeVersion = resModel.version;
            }
            
            NSLog(@"storeVersion -> %@",storeVersion);
            if ([localVersion floatValue] < [storeVersion floatValue]) {
                UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"温馨提示" message:@"有新的版本，请及时更新。" delegate:self cancelButtonTitle: nil otherButtonTitles:@"确定",@"取消",nil];
                
                [alertView show];
            }
            
        }
        
    } failure:^(AFHTTPRequestOperation * operation, NSError * error) {
        
        NSLog(@"请求失败");
        
    }];
    
    #pragma mark - UIAlertView delegate
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex{
    NSLog(@"buttonIndex -> %zi",buttonIndex);
        if(buttonIndex == 0)
        {
            [[UIApplication sharedApplication] openURL:[NSURL URLWithString:appStoreUrl]];
        }
}
    
```
