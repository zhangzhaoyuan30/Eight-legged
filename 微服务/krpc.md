# 1 负载均衡
## 1.1 权重随机
```java
protected ProviderInfo doSelect(KrpcRequest request, List<ProviderInfo> providerInfos) throws KrpcRequestException {

    int totalWeight = providerInfos.stream()
            .mapToInt(ProviderInfo::getWeight)
            .sum();

    int size = providerInfos.size();

    int random = ThreadLocalRandom.current().nextInt(totalWeight);
    for (int i = 0; i < size; i++) {
        ProviderInfo providerInfo = providerInfos.get(i);
        int weight = getWeight(providerInfo);
        random -= weight;
        if (random < 0) {
            return providerInfo;
        }
    }
    return null;
}

```
## 1.2 HRW
## 1.3 本地优先