---
layout: post
title:  "golang 开发实现K8s自定义指标自动伸缩GPU的pod副本"
date:   2023-09-06 22:13:20 +0530
categories: golang
---
背景: 这个项目负责集群deployment根据第三方返回的负载指数来配置扩缩容,作用于火山引擎的VCI容器,实现了GPU服务器弹性伸缩,可以缓解高峰期服务压力,降低用户等待时间,提高用户体验

记录一下开发主要的思路,以及遇到的坑

需求如下：自定义指定调整配置参数（执行deloyment对象，namespace, 执行间隔，扩缩容步进副本数），设置弹性容器的ttl ，防止过快被杀死，重复启动。

```
package main

import (
	"context"
	"crypto/tls"
	"encoding/json"
	"flag"
	"fmt"
	"io"
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

var ThirdUrl = "https://clouddev.xkool.ai/api/ai_backend/internal/subtask/load"
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

func myAtoi(s string) int32 {
	i, err := strconv.Atoi(s)
	if err != nil {
		panic(err)
	}
	return int32(i)
}

func scalePod(service, namespace, add, reduce, max, defaultR string, loadbalance float32, minPodAge time.Duration) BalanceAndReplicas {
	cr, sr := setReplicas(service, namespace, add, reduce, max, defaultR, loadbalance, minPodAge)
	brstruct := BalanceAndReplicas{
		LoadBalance:    loadbalance,
		Replicas:       cr,
		SourceReplicas: sr,
	}
	return brstruct
}

func setReplicas(service, namespace, add, reduce, max, defaultR string, loadbalance float32, minPodAge time.Duration) (int32, int32) {
	sr := getReplicas(service, namespace)
	var cr int32
	klog.Info(service+" 当前副本数为 ", *sr)
	ad := myAtoi(add)
	rd := myAtoi(reduce)
	ma := myAtoi(max)
	dr := myAtoi(defaultR)

	if loadbalance > 0.8 {
		if *sr <= ma {
			if *sr+ad <= ma {
				cr = *sr + ad
				klog.Info(service+" 副本数增加到 ", cr)
				return cr, *sr
			} else {
				cr = ma
				klog.Info(service+" 副本数设置为最大值 ", cr)
				return cr, *sr
			}
		}
	} else if loadbalance < 0.5 {
		// 在减少副本之前检查是否存在超过指定存活时间的Pod
		if canReduceReplicas(service, namespace, rd, dr, minPodAge) {
			if *sr >= rd+dr {
				cr = *sr - rd
				klog.Info(service+" 副本数减少到 ", cr)
				return cr, *sr
			} else {
				cr = dr
				klog.Info(service+" 当前副本数已经是最小值 ", cr)
				return cr, *sr
			}
		} else {
			cr = *sr
			return cr, *sr
		}
	} else {
		cr = *sr
		return cr, *sr
	}
	return cr, *sr
}

// 检查是否可以减少副本，确保存在超过指定存活时间的Pod
func canReduceReplicas(service, namespace string, rd, dr int32, minPodAge time.Duration) bool {
	klog.Infof("正在检查是否可以减少Pod副本数。服务: %s, 命名空间: %s, Pod最小年龄要求: %v", service, namespace, minPodAge)
	config := initClient()
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	pods, err := clientset.CoreV1().Pods(namespace).List(context.TODO(), metav1.ListOptions{
		LabelSelector: fmt.Sprintf("app=%s", service),
	})
	if err != nil {
		panic(err)
	}
	for _, pod := range pods.Items {
		age := time.Since(pod.CreationTimestamp.Time)
		klog.Infof("Pod的运行时间为 %s", age)
		if age < minPodAge {
			klog.Infof("Pod %s 的运行时间为 %s，小于最小年龄要求 %s，不允许缩减副本数。", pod.Name, age, minPodAge)
			return false
		}
	}
	klog.Info("所有Pod的运行时间都已超过最小年龄要求，允许缩减副本数。")
	return true
}

func getLoadBalance(url string) float32 {
	var loadbalance float32 = 0.6

	// 创建不验证 TLS 证书的 HTTP 客户端
	client := &http.Client{
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
		},
	}

	resp, err := client.Get(url)
	if err != nil {
		klog.Errorf("获取URL %s 失败，错误: %v", url, err)
		return loadbalance
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		klog.Errorf("请求URL %s 返回非成功状态码: %d", url, resp.StatusCode)
		return loadbalance
	}

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		klog.Errorf("读取响应体失败，错误: %v", err)
		return loadbalance
	}

	metric := &ThirdPartyMetric{}
	if err := json.Unmarshal(body, metric); err != nil {
		klog.Errorf("解析JSON失败，错误: %v", err)
		return loadbalance
	}

	loadbalance = metric.Data.LoadBalance
	klog.Infof("URL %s 负载均衡值为 %f", url, loadbalance)
	return loadbalance
}

type patch struct {
	Op    string      `json:"op"`
	Path  string      `json:"path"`
	Value interface{} `json:"value,omitempty"`
}

func updateDeploy(service, namespace string, replicas int32) error {
	rps := getReplicas(service, namespace)
	if replicas == *rps {
		klog.Info(service + " 副本数不需要更新")
		return nil
	} else if replicas < 0 {
		klog.Infof("副本数应该大于 0，当前值: %d", replicas)
		return fmt.Errorf("副本数应该大于 0")
	} else {
		var patchs []patch
		config := initClient()
		clientset, err := kubernetes.NewForConfig(config)
		if err != nil {
			panic(err.Error())
		}
		klog.Info(service + " 更新副本数 " + fmt.Sprint(replicas))
		replicaPatch := patch{Op: "replace", Path: "/spec/replicas", Value: &replicas}
		patchs = append(patchs, replicaPatch)
		patchsdata, _ := json.Marshal(patchs)
		klog.Info(string(patchsdata))
		patchresult, err := clientset.AppsV1().Deployments(namespace).Patch(context.TODO(), service, types.PatchType(types.JSONPatchType), patchsdata, metav1.PatchOptions{})
		if err != nil {
			klog.Info(err)
		} else {
			klog.Info("成功更新副本数 " + fmt.Sprint(*patchresult.Spec.Replicas))
		}
		return nil
	}
}

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

func syncScale(brchan chan BalanceAndReplicas, deploy, ns, add, reduce, max, defaultR, t, minPodAgeStr string) {
	// 解析参数
	ti, err := strconv.Atoi(t)
	if err != nil {
		klog.Errorf("参数转换错误 t=%s, 错误信息: %v", t, err)
		return
	}

	minPodAge, err := time.ParseDuration(minPodAgeStr)
	if err != nil {
		klog.Errorf("Pod最小年龄参数错误 minPodAgeStr=%s, 错误信息: %v", minPodAgeStr, err)
		return
	}
	klog.Infof("已解析Pod最小年龄为: %v", minPodAge)

	// 启动 go 协程获取负载和副本信息
	go getBalanceAndReplicasChan(deploy, ns, add, reduce, max, defaultR, brchan, time.Duration(ti)*time.Minute, minPodAge)

	// 循环接收负载均衡和副本数更新
	for br := range brchan {
		// 处理通道中的数据
		klog.Infof("\n当前负载为: %f\n源副本数为: %d\n推荐副本数为: %d\n增加步进为: %s\n减少步进为: %s\n最大副本数为: %s\n默认副本数为: %s\n执行时间间隔为: %sMin\nPod最小生命周期时间为: %s\n",
			br.LoadBalance, br.SourceReplicas, br.Replicas, add, reduce, max, defaultR, t, minPodAgeStr)

		// 更新部署的副本数
		if err := updateDeploy(deploy, ns, br.Replicas); err != nil {
			klog.Errorf("更新副本数失败: %v", err)
		}
	}
	klog.Info("负载均衡和副本数通道已关闭")
}

func getBalanceAndReplicasChan(service, namespace, addR, reduceR, max, defaultR string, brchan chan BalanceAndReplicas, duration, minPodAge time.Duration) {
	klog.Info("等待时间间隔......")

	// 使用 time.Ticker 而非 select {} 来避免不必要的 select 语句
	ticker := time.NewTicker(duration)
	defer ticker.Stop() // 确保停止定时器

	for range ticker.C {
		var url string
		if len(os.Getenv("ThirdUrl")) > 1 {
			url = os.Getenv("ThirdUrl")
			klog.Infof("使用环境变量中的ThirdUrl: %s", url)
		} else {
			url = ThirdUrl
			klog.Infof("使用默认的ThirdUrl: %s", url)
		}

		// 获取负载均衡信息并执行缩放操作
		loadbalance := getLoadBalance(url)
		b := scalePod(service, namespace, addR, reduceR, max, defaultR, loadbalance, minPodAge)

		// 将结果传递到通道
		brchan <- b
	}
}

func runWithFlag() {
	var brchan = make(chan BalanceAndReplicas)
	var deploy, ns, ad, rd, ma, dr, min, minPodAgeStr string
	flag.StringVar(&deploy, "d", "webui", "要同步副本数的部署名称。默认: webui")
	flag.StringVar(&ns, "n", "aicloud", "部署的命名空间。默认: aicloud")
	flag.StringVar(&ad, "ad", "3", "在扩容时增加的副本数。默认: 3")
	flag.StringVar(&rd, "rd", "1", "在缩容时减少的副本数。默认: 1")
	flag.StringVar(&ma, "ma", "20", "允许的最大副本数。默认: 20")
	flag.StringVar(&dr, "dr", "0", "如果不需要扩容或缩容，则使用的默认副本数。默认: 0")
	flag.StringVar(&min, "t", "5", "扩容或缩容操作之间的最小时间间隔（分钟）。默认: 5")
	flag.StringVar(&minPodAgeStr, "minPodAge", "10m", "缩容前Pod的最小生命周期时间。默认: 10m")
	flag.Parse()
	syncScale(brchan, deploy, ns, ad, rd, ma, dr, min, minPodAgeStr)
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