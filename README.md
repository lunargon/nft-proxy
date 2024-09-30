# nft-proxy
NFT Image proxy to speed up retrieval &amp; reduce reliance on RPC


1. Caches all NFTs in smaller 500x500 size
2. Provides direct mint -> image REST API
3. Provides image resizing on the fly (stretch)

### By Ntluan

## Bug
  <b>This maybe not bug</b>

  In `service/http.go`, error to get data so can be reponse with status code 500 but this is 200. 

  Current:
  ```go
  func (svc *HttpService) mediaError(c *gin.Context, err error) {
    log.Printf("Media Err: %s", err)

    c.Header("Cache-Control", "public, max=age=60") //Stop flooding
    c.Data(200, "image/jpeg", svc.defaultImage)
  }
  ```

  My suggest:
  ```go

  func (svc *HttpService) mediaError(c *gin.Context, err error) {
    log.Printf("Media Err: %s", err)

    c.Header("Cache-Control", "public, max=age=60")
    c.Data(500, "image/jpeg", svc.defaultImage)
  }
  ```

  1. Also in `service/http.go`, use `IncrementMediaRequests` without synchronization.

  Current:
  ```go
  func (svc *HttpService) showNFTMedia(c *gin.Context) {
    svc.statSvc.IncrementMediaFileRequests()
    err := svc.imgSvc.MediaFile(c, c.Param("id"))
    if err != nil {
      svc.mediaError(c, err)
      return
    }
  }
  ```

  My suggest:
  ```go
  var statSvcMutex sync.Mutex

  func (svc *HttpService) showNFTMedia(c *gin.Context) {
    statSvcMutex.Lock()
    svc.statSvc.IncrementMediaRequests()
    statSvcMutex.Unlock()

    err := svc.imgSvc.MediaFile(c, c.Param("id"))
    if err != nil {
      svc.mediaError(c, err)
      return
    }
  }
  ```
  
## Improvements:
  1. In `service/http.go`, we can make header with constant data. 

  Current:
  ```go
  c.Header("Cache-Control", "public, max=age=60") //Stop flooding

  ```

  My suggest:
  ```go
  const CacheControlHeader = "Cache-Control"
  const CacheControlValue = "public, max-age=60"

  c.Header(CacheControlHeader, CacheControlValue)
  ```

  2. In `service/solana_nft.go`, we make timeout flexible by get data from .env

  Current:
  ```go
  func (svc *SolanaImageService) Start() error {
    svc.http = &http.Client{Timeout: 5 * time.Second}

    svc.sql = svc.Service(SQLITE_SVC).(*SqliteService)
    svc.sol = svc.Service(SOLANA_SVC).(*SolanaService)
    return nil
  }
  ```

  My suggest:
  ```go
  func (svc *SolanaImageService) Start() error {
    timeout := 5 * time.Second // default timeout
    // ("HTTP_CLIENT_TIMEOUT") -> that name I give for example
    if envTimeout := os.Getenv("HTTP_CLIENT_TIMEOUT"); envTimeout != "" {
        if parsedTimeout, err := time.ParseDuration(envTimeout); err == nil {
            timeout = parsedTimeout
        }
    }
    svc.http = &http.Client{Timeout: timeout}

    svc.sql = svc.Service(SQLITE_SVC).(*SqliteService)
    svc.sol = svc.Service(SOLANA_SVC).(*SolanaService)
    return nil
  }
  ```

## Optimizations:
  In `service/solana_nft.go`, I think add `timeout` with for that

  Current:
  ```go

  func (svc *SolanaImageService) Media(key string, skipCache bool) (*nft_proxy.Media, error) {
    var media *nft_proxy.SolanaMedia
    err := svc.sql.Db().First(&media, "mint = ?", key).Error
    if err != nil || skipCache {
      log.Printf("FetchMetadata - %s err: %s", key, err)
      media, err = svc.FetchMetadata(key)
      if err != nil {
        return nil, err 
      }
    }

    return media.Media(), nil
  }
  ```

  My suggest:
  ```go
  func (svc *SolanaImageService) Media(key string, skipCache bool) (*nft_proxy.Media, error) {
    var ctx, cancel = context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    var media *nft_proxy.SolanaMedia
    err := svc.sql.Db().WithContext(ctx).First(&media, "mint = ?", key).Error
    if err != nil || skipCache {
      log.Printf("FetchMetadata - %s err: %s", key, err)
      media, err = svc.FetchMetadata(key)
      if err != nil {
        return nil, err 
      }
    }

    return media.Media(), nil
  }
  ```
