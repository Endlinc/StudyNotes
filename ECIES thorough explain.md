## ECIES thorough explain

### Abstract

ECIES的全称是椭圆曲线集成加密方案，以下简称ECIES。其中集成加密方案是指混合加密方案，集合了公钥加密系统的便利性和对称密钥加密系统的高效。ECIES就是集成加密方案的一个主要变种，另一个是离散对数集成加密方案。ECIES中的对称加密算法和MAC算法可以自由选择，也给与了一定的可定制性。

### Math Explain

ECIES同其他公钥加密系统一样，需要持有一对公私钥。而密钥生成是通过在一个椭圆曲线中定义两点`P & R`，定义出一条穿过椭圆曲线的直线。如函数`y=x^3+ax+b`的函数图像，下图中通过定义椭圆曲线中P、R两点我们定义出一条直线，直线的斜率就是我们的私钥，P点我们定义为这段曲线的开始点。小节，生成ECIES的公私密钥有2个参数，私钥是直线的斜率，以下称`dA`；P点是我们定义的曲线段开始点，以下称为`G`。

![Diagram graphs the elliptic curve equation y=x³ + ax + b.](https://avinetworks.com/wp-content/uploads/2020/02/elliptic-curve-cryptography-diagram.png)

我们持有了密钥`dA`和`G`点值，当前要生成公钥`QA`，
$$
Q_A=d_A\times G
$$
现在我们得到的公钥由密钥和`G`点值做标量乘法得到。

#### Encryption

要加密一条消息m，我们需要下列步骤：

首先我们需要选择一个随机数r，然后计算R值和S值，R值用来返回给解密方、而S值用来生成密钥：
$$
\begin{align*}
R&=r\times G\ \ where\ \ r\in [1, n-1]\\
S&=r\times Q_A
\end{align*}
$$
R值返回给解密方，于是计算得到加秘方用于加密的密钥S：
$$
\begin{align*}
S&=d_A\times R\\
S&=d_A\times (r\times G)\\
S&=r\times (d_A\times G)\\
S&=r\times Q_A
\end{align*}
$$
我们知道S是一个点，所以我们取Sx坐标，使用*密钥派生函数*(使用伪随机函数从诸如主密钥或密码的秘密值中派生出一个或多个密钥)导出对称密钥和MAC钥：
$$
k_E||k_M=KDF(S_x)
$$
使用对称密钥加密信息m：
$$
c=E(k_E;m)
$$
使用MAC钥计算密文tag：
$$
d=MAC(k_M;c)
$$
ECIES最后输出为：
$$
R||c||d
$$

#### Decryption

解密密文`R||c||d`我们需要：

从密钥R中导出加密用到的共享密钥S：
$$
S=P_X\ \  where\ \ P=(P_x,P_y)=r\times Q_A
$$
在介绍加密时已经介绍过是如何从R推导出S。

获取到S过后，同样按照加密的过程，使用*密钥派生函数*（KDF），推导出加密使用的密钥
$$
k_E||k_M=KDF(S_x)
$$
使用MAC检查tag是否和d相同，拒绝继续解密如果：
$$
d\ne MAC(k_M;c)
$$
使用对称加密算法解密出信息m：
$$
m=E^{-1}(k_E;c)
$$

### Implementation

实现ECIES的端到端流程，包括了不同的步骤。包括了公私钥生成、临时密钥对生成、密钥协议（KA）、KDF、加密、tag过程，如下图：

![img](https://60896510-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-LhlOQMrG9bRiqWpegM0%2Fuploads%2Fgit-blob-19769dbf2cf56f28a6fab0b90fb10a8ab1506874%2Fecies.png?alt=media)

#### Key pair generation

公私钥生成，首先要提供一条椭圆曲线，在go中用`elliptic.Curve`描述曲线对象，主要定义了数学上的P点、G点和N值：

```go
// CurveParams contains the parameters of an elliptic curve and also provides
// a generic, non-constant time implementation of Curve.
type CurveParams struct {
	P       *big.Int // the order of the underlying field
	N       *big.Int // the order of the base point
	B       *big.Int // the constant of the curve equation
	Gx, Gy  *big.Int // (x,y) of the base point
	BitSize int      // the size of the underlying field
	Name    string   // the canonical name of the curve
}
```

其次要提供一条随机的直线，为了方便使用，记录直线的斜率。在go中，我们使用安全随机器生成随机数表示这样一条曲线。为了保证两条线存在焦点，要保证这个随机数在范围内。最后我们得到私钥（随机数）、G（交点x、y坐标值）：

```go
// GenerateKey returns a public/private key pair. The private key is
// generated using the given reader, which must return random data.
func GenerateKey(curve Curve, rand io.Reader) (priv []byte, x, y *big.Int, err error) {
	N := curve.Params().N
	bitSize := N.BitLen()
	byteLen := (bitSize + 7) / 8
	priv = make([]byte, byteLen)

	for x == nil {
		_, err = io.ReadFull(rand, priv)
		if err != nil {
			return
		}
		// We have to mask off any excess bits in the case that the size of the
		// underlying field is not a whole number of bytes.
		priv[0] &= mask[bitSize%8]
		// This is because, in tests, rand will return all zeros and we don't
		// want to get the point at infinity and loop forever.
		priv[1] ^= 0x42

		// If the scalar is out of range, sample another random number.
		if new(big.Int).SetBytes(priv).Cmp(N) >= 0 {
			continue
		}

		x, y = curve.ScalarBaseMult(priv)
	}
	return
}
```

同样的，要生成一条临时密钥对，我们重复上述生成公私钥的步骤。

#### Key Agreement

接下来需要定义KA，生成我们的共享密钥，这里我们使用ECDH KA方法，前面介绍的生成公钥方程：
$$
Q_A=d_A\times G
$$
代码实现如下：

```go
// ECDH key agreement method used to establish secret keys for encryption.
func (prv *PrivateKey) GenerateShared(pub *PublicKey, skLen, macLen int) (sk []byte, err error) {
	if prv.PublicKey.Curve != pub.Curve {
		return nil, ErrInvalidCurve
	}
	if skLen+macLen > MaxSharedKeyLength(pub) {
		return nil, ErrSharedKeyTooBig
	}

	x, _ := pub.Curve.ScalarMult(pub.X, pub.Y, prv.D.Bytes())
	if x == nil {
		return nil, ErrSharedKeyIsPointAtInfinity
	}

	sk = make([]byte, skLen+macLen)
	skBytes := x.Bytes()
	copy(sk[len(sk)-len(skBytes):], skBytes)
	return sk, nil
}
```

#### KDF

获取到共享密钥后，我们需要使用KDF，推到出对称加密密钥和MAC钥，这里使用的是[NIST SP 800-56 Concatenation Key Derivation Function][1]（参照5.8.1）：

```go
// NIST SP 800-56 Concatenation Key Derivation Function (see section 5.8.1).
func concatKDF(hash hash.Hash, z, s1 []byte, kdLen int) (k []byte, err error) {
	if s1 == nil {
		s1 = make([]byte, 0)
	}

	reps := ((kdLen + 7) * 8) / (hash.BlockSize() * 8)
	if big.NewInt(int64(reps)).Cmp(big2To32M1) > 0 {
		fmt.Println(big2To32M1)
		return nil, ErrKeyDataTooLong
	}

	counter := []byte{0, 0, 0, 1}
	k = make([]byte, 0)

	for i := 0; i <= reps; i++ {
		hash.Write(counter)
		hash.Write(z)
		hash.Write(s1)
		k = append(k, hash.Sum(nil)...)
		hash.Reset()
		incCounter(counter)
	}

	k = k[:kdLen]
	return
}
```

生成了对称密钥和MAC钥后，分割成对应密钥：

```go
K, err := concatKDF(hash, z, s1, params.KeyLen+params.KeyLen)
if err != nil {
	return
}
Ke := K[:params.KeyLen]
Km := K[params.KeyLen:]
hash.Write(Km)
Km = hash.Sum(nil)
```

#### Encrypt

加密可以使用不同对称加密算法，在实现时，我们使用了不同的公钥椭圆曲线类型对应了不同的加密算法和哈希算法：

```go
var paramsFromCurve = map[elliptic.Curve]*ECIESParams{
	elliptic.P384():  ECIES_AES256_SHA384,
	elliptic.P521():  ECIES_AES256_SHA512,
}
var (
	ECIES_AES256_SHA384 = &ECIESParams{
		Hash:      sha512.New384,
		hashAlgo:  crypto.SHA384,
		Cipher:    aes.NewCipher,
		BlockSize: aes.BlockSize,
		KeyLen:    32,
	}

	ECIES_AES256_SHA512 = &ECIESParams{
		Hash:      sha512.New,
		hashAlgo:  crypto.SHA512,
		Cipher:    aes.NewCipher,
		BlockSize: aes.BlockSize,
		KeyLen:    32,
	}
)
```

这里使用了AES256 CTR模式进行加密：

```go
// symEncrypt carries out CTR encryption using the block cipher specified in the
// parameters.
func symEncrypt(rand io.Reader, params *ECIESParams, key, m []byte) (ct []byte, err error) {
	c, err := params.Cipher(key)
	if err != nil {
		return
	}

	iv, err := generateIV(params, rand)
	if err != nil {
		return
	}
	ctr := cipher.NewCTR(c, iv)

	ct = make([]byte, len(m)+params.BlockSize)
	copy(ct, iv)
	ctr.XORKeyStream(ct[params.BlockSize:], m)
	return
}
```

#### Tag

加密后，计算密文tag，保证密文不被篡改：

```
// messageTag computes the MAC of a message (called the tag) as per
// SEC 1, 3.5.
func messageTag(hash func() hash.Hash, km, msg, shared []byte) []byte {
	mac := hmac.New(hash, km)
	mac.Write(msg)
	mac.Write(shared)
	tag := mac.Sum(nil)
	return tag
}
```

#### 生成输出

上面我们已经生成了密文和密文的tag，接下来我们要使用公钥生成R值，再拼接这三个的值：

```go
Rb := elliptic.Marshal(pub.Curve, R.PublicKey.X, R.PublicKey.Y)
ct = make([]byte, len(Rb)+len(em)+len(d))
copy(ct, Rb)
copy(ct[len(Rb):], em)
copy(ct[len(Rb)+len(em):], d)
```



- [1]: <https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-56ar.pdf> "Recommendation for Pair-Wise Key Establishment Schemes Using Discrete Logarithm Cryptography"

