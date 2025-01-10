---
layout: post
title:  "golang 开发实现K8s自定义指标自动伸缩GPU的pod副本"
date:   2023-09-06 22:13:20 +0530
categories: golang
---
背景: 这个项目负责集群deployment根据第三方返回的负载指数来配置扩缩容,作用于火山引擎的VCI容器,实现了GPU服务器弹性伸缩,可以缓解高峰期服务压力,降低用户等待时间,提高用户体验

记录一下开发主要的思路,以及遇到的坑

```
package main

import (
	"context"
	"encoding/json"
	"flag"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"strconv"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/klog/v2"
)

var ThirdUrl = "http://***/internal/subtask/load"
var KubeConfigPath = "./config"

type Data struct {
	LoadBalance float32 `json:"load_balance"`
}

type ThirdPartyMetric struct {
	Data Data
}

type BalanceAndReplicas struct {
	LoadBalance    float32
	Replicas       int32
	SourceReplicas int32
}

// string转换成int类型
func myAtoi(s string) int32 {
	i, err := strconv.Atoi(s)
	if err != nil {
		panic(err)
	}
	return int32(i)
}

// 根据负载均衡情况调整副本数
func scalePod(service, namespace, add, reduce, max, defaultR string, loadbalance float32) BalanceAndReplicas {
	cr, sr := setReplicas(service, namespace, add, reduce, max, defaultR, loadbalance)
	brstruct := BalanceAndReplicas{
		LoadBalance:    loadbalance,
		Replicas:       cr,
		SourceReplicas: sr,
	}
	return brstruct
}

// 根据负载均衡计算副本数
func setReplicas(service, namespace, add, reduce, max, defaultR string, loadbalance float32) (int32, int32) {
	sr := getReplicas(service, namespace)
	var cr int32
	klog.Info(service+" current replicas is ", *sr)
	ad := myAtoi(add)
	rd := myAtoi(reduce)
	ma := myAtoi(max)
	dr := myAtoi(defaultR)
	if loadbalance > 0.8 {
		// 不允许超过最大ma的情况下,负载均衡大于0.8,需要增加副本,默认增加2个
		if *sr <= int32(ma) {
			if *sr+int32(ad) <= int32(ma) {
				cr := *sr + int32(ad)
				klog.Info(service+" replicas increased to ", cr)
				return cr, *sr
			} else {
				cr = int32(ma)
				klog.Info(service+" replicas set to max ", cr)
				return cr, *sr
			}
		}
	} else if loadbalance < 0.5 {
		// 负载均衡小于0.5,需要根据源副本数减少副本,默认减少1个
		if *sr >= int32(rd)+int32(dr) {
			cr := *sr - int32(rd)
			klog.Info(service+" replicas reduced to ", cr)
			return cr, *sr
		} else {
			cr := int32(dr)
			klog.Info(service+" current replicas is already at minimum ", cr)
			return cr, *sr
		}
	} else {
		cr = *sr
		return cr, *sr
	}
	return cr, *sr
}

// 获取负载
func getLoadBalance(url string) float32 {
	var loadbalance float32
	resp, err := http.Get(url)
	if err != nil {
		klog.Errorf("failed to get url %s, error: %v", url, err)
		return 0.6	// 当获取负载超时的时候,0.6的值确证不修改副本
	}
	defer resp.Body.Close()
	body, _ := ioutil.ReadAll(resp.Body)
	klog.Infof(" third party metric" + string(body))
	metric := &ThirdPartyMetric{}
	json.Unmarshal(body, metric)
	loadbalance = metric.Data.LoadBalance
	klog.Infof(url+" load balance is %f", loadbalance)
	return loadbalance
}

type patch struct {
	Op    string      `json:"op"`
	Path  string      `json:"path"`
	Value interface{} `json:"value,omitempty"`
}

// 更新deployment的replicas数量
func updateDeploy(service, namespace string, replicas int32) error {
	rps := getReplicas(service, namespace)
	// 如果副本数不变则不需要更新
	if replicas == *rps {
		klog.Info(service + " replicas does not need to be updated")
		return nil
		// 副本数小于零则报错
	} else if replicas < 0 {
		klog.Infof("Replicas number should be greater than 0, current value: %d", replicas)
		return fmt.Errorf("replicas number should be greater than 0")
	} else {
		var patchs []patch
		config := initClient()
		clientset, err := kubernetes.NewForConfig(config)
		if err != nil {
			panic(err.Error())
		}
		klog.Info(service + " update replicas " + fmt.Sprint(replicas))
		replicaPatch := patch{Op: "replace", Path: "/spec/replicas", Value: &replicas}
		patchs = append(patchs, replicaPatch)
		patchsdata, _ := json.Marshal(patchs)
		klog.Info(string(patchsdata))
		patchresult, err := clientset.AppsV1().Deployments(namespace).Patch(context.TODO(), service, types.PatchType(types.JSONPatchType), patchsdata, metav1.PatchOptions{})
		if err != nil {
			klog.Info(err)
		} else {
			klog.Info("Successful update replicas " + fmt.Sprint(*patchresult.Spec.Replicas))
		}
		return nil
	}
}

// 获取副本数
func getReplicas(service, namespace string) *int32 {
	config := initClient()
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	deployment, err := clientset.AppsV1().Deployments(namespace).Get(context.TODO(), service, metav1.GetOptions{})
	if err != nil {
		panic(err)
	}
	r := deployment.Spec.Replicas

	return r
}

// 初始化K8s客户端
func initClient() *rest.Config {
	if len(os.Getenv("KUBECONFIG_PATH")) > 1 {
		KubeConfigPath := os.Getenv("KUBECONFIG_PATH")
		config, err := clientcmd.BuildConfigFromFlags("", KubeConfigPath)
		if err != nil {
			panic(err)
		}
		return config
	} else {
		config, err := clientcmd.BuildConfigFromFlags("", KubeConfigPath)
		if err != nil {
			panic(err)
		}
		return config
	}
}

// 同步扩缩容
func syncScale(brchan chan BalanceAndReplicas, deploy, ns, add, reduce, max, defaultR, t string) {
	// Get a channel to receive messages on
	ti, err := strconv.Atoi(t)
	if err != nil {
		panic(err)
	}
	go getBalanceAndReplicasChan(deploy, ns, add, reduce, max, defaultR, brchan, time.Duration(ti)*time.Minute)
	for {
		select {
		case br, ok := <-brchan:
			if !ok {
				klog.Info("Balance and replicas channel closed")
			}
			klog.Infof("\n当前负载为: %f\n源副本数为: %d\n推荐副本数为: %d\n增加步进为: %s\n减少步进为: %s\n最大副本数为: %s\n默认副本数为: %s\n执行时间间隔为: %sMin\n", br.LoadBalance, br.SourceReplicas, br.Replicas, add, reduce, max, defaultR, t)
			// Sync replicas to Kubernetes
			err := updateDeploy(deploy, ns, br.Replicas)
			if err != nil {
				klog.Errorf("Failed to update replicas: %v", err)
			}
		}
	}
}

func getBalanceAndReplicasChan(service, namespace, addR, reduceR, max, defaultR string, brchan chan BalanceAndReplicas, duration time.Duration) {
	//获取负载参数管道,默认每隔5分钟获取一次负载均衡参数
	klog.Info("Waiting for time duration ......")
	for {
		select {
		case <-time.After(duration):
			//优先获取环境变量中的ThirdUrl配置
			var url string
			if len(os.Getenv("ThirdUrl")) > 1 {
				url = os.Getenv("ThirdUrl")
				klog.Infof("Using ThirdUrl from environment variable: %s", url)
				loadbalance := getLoadBalance(url)
				b := scalePod(service, namespace, addR, reduceR, max, defaultR, loadbalance)
				brchan <- b
			} else {
				klog.Infof("Using  Default ThirdUrl: %s", ThirdUrl)
				loadbalance := getLoadBalance(ThirdUrl)
				b := scalePod(service, namespace, addR, reduceR, max, defaultR, loadbalance)
				brchan <- b
			}
		}
	}
}

// 传参处理函数
func runWithFlag() {
	var brchan = make(chan BalanceAndReplicas)
	var deploy, ns, ad, rd, ma, dr, min string
	flag.StringVar(&deploy, "d", "webui", "The name of the deployment to sync replicas for. default: webui")
	flag.StringVar(&ns, "n", "aicloud", "The namespace of the deployment. default: aicloud")
	flag.StringVar(&ad, "ad", "3", "The number of replicas to add on scale up. default: 3")
	flag.StringVar(&rd, "rd", "1", "The number of replicas to reduce on scale down. default: 1")
	flag.StringVar(&ma, "ma", "20", "The maximum number of replicas allowed. default: 20")
	flag.StringVar(&dr, "dr", "0", "The default number of replicas if no scale is needed. default: 0")
	flag.StringVar(&min, "t", "5", "The minimum number of minutes between scaling operations. deault: 5")
	flag.Parse()
	syncScale(brchan, deploy, ns, ad, rd, ma, dr, min)
}

func main() {
	runWithFlag()
}
```

主函数逻辑: 获取自定义接口函数的负载均衡值, 我这里使用的后端的返回负载值的接口,然后根据负载均衡值计算出副本数,最后更新副本数,同时要实现适当调整时间间隔, 避免一直处于扩缩容的触发状态,达到自动扩缩容的效果.同时灵活调整副本数从而提升为命令行参数,方便动态调整步进.

dockerfile 构建镜像
```
ARG IMAGE=alpine:3.12

FROM golang:1.20-alpine as builder
WORKDIR ${GOPATH}/src/cusme
ENV  GOPROXY https://goproxy.cn,direct
COPY . ./
RUN CGO_ENABLED=0 GOOS=linux go mod tidy && go build -o /usr/bin/cusme

FROM ${IMAGE}
WORKDIR /app
RUN  sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories \
      &&  apk add --no-cache bash openssl curl
COPY --from=builder /usr/bin/cusme /app
COPY . /app
ENTRYPOINT ["/app/cusme"]
```

k8s部署文件
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cusme
  namespace: sre
spec:
  selector:
    matchLabels:
      app: cusme
  template:
    metadata:
      labels:
        app: cusme
    spec:
      containers:
      - env:
        - name: ThirdUrl
          value: "http://***/internal/subtask/load"
        image: cusme:1.0.0
        args: ["-t", "1", "-n", "***", "-d", "***", "-ad", "3", "-rd", "1", "-dr", "0"]
        imagePullPolicy: Always
        name: cusme-clouddev
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: oci-secret
      restartPolicy: Always
```
cusme 为控制器, 副本数为1, 镜像为构建好的镜像, 启动参数为自定义参数, 启动命令为自定义参数, 启动策略为Always,保证控制器的正常运行.