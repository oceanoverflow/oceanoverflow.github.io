  由于某些特定加密算法的要求，例如AES中的CBC，要求被加密的数据是特定块大小的整数倍，例如在AES中一个块大小即BlockSize的大小为16字节，故被加密数据的
长度`len(src) % BlockSize == 0`，但现实生活中要求加密的数据长度往往是随机的，所以需要有一种可靠高效的数据填充方法（padding）。

  现在我们的问题是这样的，一个不定长度的数据作为加密的输入，我们需要把它填充到固定的长度，即满足`len(src) % BlockSize == 0`的条件，并且在解密时
可以获得原来的数据。

  现在的前提是BlockSize为16字节，输入长度随机。流程大致经过下面几步，填充=>加密=>网络传输=>解密=>去填充。问题的难点就是如何保证去填充之后可以获取
原来填充之前的数据。

  我们的初始数据即待加密数据的格式如下
  
  |XX16bytesXX|XX16bytesXX|…….|XX16bytesXX|XX?bytesXX|
  
  填充后得到的数据格式为如下。
  
  |XX16bytesXX|XX16bytesXX|…….|XX16bytesXX|XX16bytesXX|
  
  这里我们使用 |XX16bytesXX| 表示一段16字节的随机数据。也就是说我们需要把最后的?bytes填充到固定长度的16bytes，这里？的取值范围是0到15。
现在我们就来引入PKCS7这种填充标准。

  PKCS7填充的做法是这样的，首先它会确定需要被填充的字节的长度，也就是上述所说的?bytes的那一段字节。
```golang
padding := aes.BlockSize - len(src)%aes.BlockSize
```

  上面的代码段非常好理解，首先对待加密的数据长度对BlockSize取模，获得我们的?bytes，再利用它与BlockSize的差值获得我们需要填充的长度值，
这里的padding就是一个int值。

  下面就是真正填充的一个逻辑了。
  
```golang
padtext := bytes.Repeat([]byte{byte(padding)}, padding)
```

  这一行代码乍看起来有点复杂，我们一步一步来看，它要做的一件事情就是获得一个用于填充的文字，而这个文字就是利用我们上述获得的padding（注意它是一个int）
转化成一个byte后重复padding次。

  也就是说假设我们的padding是7，也就是需要填充7字节的信息，那我们的padtext也就是我们填充的文字就是 7 7 7 7 7 7 7，当然我们这里的每个7都是一个byte。
  
  以此类推，同理我们padtext里面的每一个数字就是一个byte，6 6 6 6 6 6 应该理解成byte(6) byte(6) byte(6) byte(6) byte(6) byte(6)。
  
  获得了我们的填充信息后，我们将原数据与我们的填充信息相连接就行了，下面就是我们完整填充过程的函数，非常简单。
  
```golang
func PKCS7Padding(src []byte) []byte {
    padding := aes.BlockSize - len(src)%aes.BlockSize
    padtext := bytes.Repeat([]byte{byte(padding)}, padding)
    return append(src, padtext...)
}
```

  理解了PKCS7中填充的标准之后，我们再来看一下如何去填充，去填充的过程也非常简单，但是在工程代码中实现还需要考虑一些特殊的边界情况。
  
	下面直接给出去填充的代码。
  
```golang
func PKCS7Unpadding(src []byte) ([]byte, error) {
    length := len(src)
    unpadding := int(src[length - 1])
	
    if unpadding > aes.BlockSize || unpadding == 0 {
    return nil, fmt.Errorf("invalid unpadding")
    }
	  
    pad := src[len(src)-unpadding:]
    for i := 0; i < unpadding; i++ {
        if pad[i] != byte(unpadding) {
            return nil, fmt.Errorf("invalid padding")
        }
    }

    return src[:(len(src)-unpadding)], nil
}	
```
  这里我们需要去填充的为src，讲src的最后一个byte取出再转换为int，这个数值就是我们之前填充字节的个数，记为unpadding，如果unpadding的值大于
BlockSize或者出现unpadding为0的情况，这种情况是一场的因为我们填充的时候就将padding值约束在[1, BlockSize-1)，所以出现不符合要求的padding值都
将返回异常。

  然后将数组末尾取出unpadding个字节，将unpadding个字节分别与byte(unpadding)这个字节进行比较，如果不想等，则返回异常。

  最后如果一切都没有问题，则返回去除padding个字节后的byte数组。


