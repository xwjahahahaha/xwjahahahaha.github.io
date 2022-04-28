---
title: Kubeadm的解读
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-16 20:15:40
---

# 一、kubeadm工具简介

https://github.com/kubernetes/kubeadm

* 起源：为了增强用户体验，社区希望有一个类似于Swarm的`init`、`join`的简单的方式去安装k8s
* 作用：官方提供的快速安装部署一个k8s集群的引导工具

<!-- more -->

* 注意：kubeadm只关心引导程序，而不关心其他的一些必要机器配置

  * 网络插件要求，CNI插件容器连接到Linux网桥(将`net/bridge/bridge-nf-call-iptablessysctl`设置为 1，以确保`iptables`代理正常运行) 

  * 容器运行时选择(默认情况下，`Kubernetes`使用 容器运行时接口 (CRI)与选择的容器运行时交互)

  * kubeadm不会安装或管理kubelet或kubectl，所以你需要确保他们想要kubeadm 安装适合你的Kubernetes控制平面的版本匹配

  * 配置cgroup驱动程序，需要匹配容器运行时和kubelet cgroup驱动程序，否则kubelet进程将失败

    比如默认如果使用kubeadm安装的集群，`cgroupdriver: systemd`那么如果你使用docker容器运行时，需要在守护进程 配置`cgroupdriver=systemd`,否者这会导致kubelet进程失败

# 二、kubeadm的整体流程

## init和join命令整体流程

![kubeadm](http://xwjpics.gumptlu.work/qinniu_uPic/kubeadm.png)

# 三、源码学习

> * 代码所在地址：https://github.com/kubernetes/kubernetes/tree/master/cmd/kubeadm

kubeadm命令行工具使用规则：

![image-20220419152328071](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220419152328071.png)

## 1. 命令传入

主函数生成根命令（cobra）并执行:

生成根命令以及其他子命令：

```go
// NewKubeadmCommand returns cobra.Command to run kubeadm command
func NewKubeadmCommand(in io.Reader, out, err io.Writer) *cobra.Command {
   var rootfsPath string
   // 根命令
   cmds := &cobra.Command{
      Use:   "kubeadm",
      Short: "kubeadm: easily bootstrap a secure Kubernetes cluster",
      Long: dedent.Dedent(`

             ┌──────────────────────────────────────────────────────────┐
             │ KUBEADM                                                  │
             │ Easily bootstrap a secure Kubernetes cluster             │
             │                                                          │
             │ Please give us feedback at:                              │
             │ https://github.com/kubernetes/kubeadm/issues             │
             └──────────────────────────────────────────────────────────┘
							....

      `),
      SilenceErrors: true,
      SilenceUsage:  true,
      PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
         if rootfsPath != "" {
            if err := kubeadmutil.Chroot(rootfsPath); err != nil {
               return err
            }
         }
         return nil
      },
   }

   cmds.ResetFlags()

   // 添加所有子命令
   cmds.AddCommand(newCmdCertsUtility(out))
   cmds.AddCommand(newCmdCompletion(out, ""))
   cmds.AddCommand(newCmdConfig(out))
   cmds.AddCommand(newCmdInit(out, nil))
   cmds.AddCommand(newCmdJoin(out, nil))
   cmds.AddCommand(newCmdReset(in, out, nil))
   cmds.AddCommand(newCmdVersion(out))
   cmds.AddCommand(newCmdToken(out, err))
   cmds.AddCommand(upgrade.NewCmdUpgrade(out))
   cmds.AddCommand(alpha.NewCmdAlpha())
   options.AddKubeadmOtherFlags(cmds.PersistentFlags(), &rootfsPath)
   cmds.AddCommand(newCmdKubeConfigUtility(out))

   return cmds
}
```

## 2. reset命令

命令行逻辑代码：

```go
// newCmdReset returns the "kubeadm reset" command
func newCmdReset(in io.Reader, out io.Writer, resetOptions *resetOptions) *cobra.Command {
	if resetOptions == nil {
		resetOptions = newResetOptions()
	}
	resetRunner := workflow.NewRunner()

	cmd := &cobra.Command{
		Use:   "reset",
		Short: "Performs a best effort revert of changes made to this host by 'kubeadm init' or 'kubeadm join'",
		RunE: func(cmd *cobra.Command, args []string) error {
			c, err := resetRunner.InitData(args)
			if err != nil {
				return err
			}

			err = resetRunner.Run(args)
			if err != nil {
				return err
			}

			// Then clean contents from the stateful kubelet, etcd and cni directories
			data := c.(*resetData)
			cleanDirs(data)

			// output help text instructing user how to remove cni folders
			fmt.Print(cniCleanupInstructions)
			// Output help text instructing user how to remove iptables rules
			fmt.Print(iptablesCleanupInstructions)
			return nil
		},
	}

	AddResetFlags(cmd.Flags(), resetOptions)

	// initialize the workflow runner with the list of phases
	resetRunner.AppendPhase(phases.NewPreflightPhase())			// 对应preflight运行reset前检查
	resetRunner.AppendPhase(phases.NewRemoveETCDMemberPhase())	// 删除本地ETCD集群成员
	resetRunner.AppendPhase(phases.NewCleanupNodePhase())		// 对应执行cleanup node命令

	// sets the data builder function, that will be used by the runner
	// both when running the entire workflow or single phases
	resetRunner.SetDataInitializer(func(cmd *cobra.Command, args []string) (workflow.RunData, error) {
		return newResetData(cmd, resetOptions, in, out)
	})

	// binds the Runner to kubeadm init command by altering
	// command help, adding --skip-phases flag and by adding phases subcommands
	resetRunner.BindToCommand(cmd)

	return cmd
}
```

### NewPreflightPhase函数

首先运行一下`kubeadm reset`查看具体做了哪些事：

```shell
# kubeadm reset
[reset] Reading configuration from the cluster...
[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm ...'
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be rev  
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
[reset] Removing info for node "master" from the ConfigMap "kubeadm-config" in the "kube-s  
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernet  
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kub  
[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/...]
```

具体实现代码`app/phases/reset/prefight.go`：

```go
// NewPreflightPhase creates a kubeadm workflow phase implements preflight checks for reset
func NewPreflightPhase() workflow.Phase {
	return workflow.Phase{
		Name:    "preflight",
		Aliases: []string{"pre-flight"},
		Short:   "Run reset pre-flight checks",
		Long:    "Run pre-flight checks for kubeadm reset.",
		Run:     runPreflight,
		InheritFlags: []string{
			// 预检查的选项
			options.IgnorePreflightErrors,		// 忽略预检查时的错误
			options.ForceReset,
		},
	}
}

// runPreflight executes preflight checks logic.
func runPreflight(c workflow.RunData) error {
	r, ok := c.(resetData)
	if !ok {
		return errors.New("preflight phase invoked with an invalid data struct")
	}

	if !r.ForceReset() {		// 判断是否是强制reset
		// 再次提示确认
		klog.Warning("[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.")
		fmt.Print("[reset] Are you sure you want to proceed? [y/N]: ")
		s := bufio.NewScanner(r.InputReader())
		s.Scan()
		if err := s.Err(); err != nil {
			return err
		}
		if strings.ToLower(s.Text()) != "y" {
			return errors.New("aborted reset operation")
		}
	}

	fmt.Println("[preflight] Running pre-flight checks")
	// 执行预检查
	return preflight.RunRootCheckOnly(r.IgnorePreflightErrors())
}
```

预检查只检查了当前用户是否是管理员

```go
// RunRootCheckOnly initializes checks slice of structs and call RunChecks
func RunRootCheckOnly(ignorePreflightErrors sets.String) error {
	// 生成检查
	checks := []Checker{
		// IsPrivilegedUserCheck用于检查用户是否是管理员(linux - root, windows - Administrator)
		IsPrivilegedUserCheck{},		
	}

	return RunChecks(checks, os.Stderr, ignorePreflightErrors)
}
```

总结：

* `NewPreflightPhase`函数只是在reset具体操作之前的root权限检查，并且附带一个再次确认的交互，所以此时还未执行到真正的reset命令逻辑

* `NewPreflightPhase`函数在整个`kubeadm`源码中有四处不同的同名函数（逻辑不同）：

  <img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220423123620458.png" alt="image-20220423123620458" style="zoom:50%;" />

  * 第一种就是init初始化控制平台的检查
  * 第二种是join加入节点的检查
  * 第三种就是reset的卸载节点的检查
  * 第四种就是upgrade更新集群节点的检查

### NewRemoveETCDMemberPhase函数

根据etcd清单文件 -> 读取etcd静态pod信息 -> 获取etcd-data挂载的目录 -> 删除etcd的数据目录(本地)

```go
// NewRemoveETCDMemberPhase creates a kubeadm workflow phase for remove-etcd-member
func NewRemoveETCDMemberPhase() workflow.Phase {
	return workflow.Phase{
		Name:  "remove-etcd-member",
		Short: "Remove a local etcd member.",
		Long:  "Remove a local etcd member for a control plane node.",
		Run:   runRemoveETCDMemberPhase,
		InheritFlags: []string{
			options.KubeconfigPath,
		},
	}
}

func runRemoveETCDMemberPhase(c workflow.RunData) error {
	r, ok := c.(resetData)
	if !ok {
		return errors.New("remove-etcd-member-phase phase invoked with an invalid data struct")
	}
	cfg := r.Cfg()

	// Only clear etcd data when using local etcd.
	klog.V(1).Infoln("[reset] Checking for etcd config")
	etcdManifestPath := filepath.Join(kubeadmconstants.KubernetesDir, kubeadmconstants.ManifestsSubDirName, "etcd.yaml")
	// 获取etcd数据的目录
	etcdDataDir, err := getEtcdDataDir(etcdManifestPath, cfg)
	if err == nil {
    // 删除
		r.AddDirsToClean(etcdDataDir)
		if cfg != nil {
			if !r.DryRun() {
				if err := etcdphase.RemoveStackedEtcdMemberFromCluster(r.Client(), cfg); err != nil {
					klog.Warningf("[reset] Failed to remove etcd member: %v, please manually remove this etcd member using etcdctl", err)
				}
			} else {
				fmt.Println("[reset] Would remove the etcd member on this node from the etcd cluster")
			}
		}
	} else {
		fmt.Println("[reset] No etcd config found. Assuming external etcd")
		fmt.Println("[reset] Please, manually reset etcd to prevent further issues")
	}

	return nil
}

func getEtcdDataDir(manifestPath string, cfg *kubeadmapi.InitConfiguration) (string, error) {
	const etcdVolumeName = "etcd-data"
	var dataDir string

	if cfg != nil && cfg.Etcd.Local != nil {
		return cfg.Etcd.Local.DataDir, nil
	}
	klog.Warningln("[reset] No kubeadm config, using etcd pod spec to get data directory")
	
	// 通过etcd的清单获取到etcd的静态pod信息
	etcdPod, err := utilstaticpod.ReadStaticPodFromDisk(manifestPath)
	if err != nil {
		return "", err
	}
	
	// 从mount挂载目录中获取到etcd-data的目录,并返回
	for _, volumeMount := range etcdPod.Spec.Volumes {
		if volumeMount.Name == etcdVolumeName {
			dataDir = volumeMount.HostPath.Path
			break
		}
	}
	if dataDir == "" {
		return dataDir, errors.New("invalid etcd pod manifest")
	}
	return dataDir, nil
}
```

### NewCleanupNodePhase函数

核心就是通过init系统进程（例如`systemctl`、`openrc`）停止kubelet服务，然后删除一系列的目录

```go
func runCleanupNode(c workflow.RunData) error {
	r, ok := c.(resetData)
	if !ok {
		return errors.New("cleanup-node phase invoked with an invalid data struct")
	}
	certsDir := r.CertificatesDir()

	// Try to stop the kubelet service
	klog.V(1).Infoln("[reset] Getting init system")
	// 根据不同的操作系统(unix、windows)获取init系统进程（例如systemctl），停止kubelet服务
	initSystem, err := initsystem.GetInitSystem()
	if err != nil {
		klog.Warningln("[reset] The kubelet service could not be stopped by kubeadm. Unable to detect a supported init system!")
		klog.Warningln("[reset] Please ensure kubelet is stopped manually")
	} else {
		if !r.DryRun() {
			fmt.Println("[reset] Stopping the kubelet service")
			// 停止服务
			if err := initSystem.ServiceStop("kubelet"); err != nil {
				klog.Warningf("[reset] The kubelet service could not be stopped by kubeadm: [%v]\n", err)
				klog.Warningln("[reset] Please ensure kubelet is stopped manually")
			}
		} else {
			fmt.Println("[reset] Would stop the kubelet service")
		}
	}

	if !r.DryRun() {
		// 解挂载kubelet的运行目录
		// Try to unmount mounted directories under kubeadmconstants.KubeletRunDirectory in order to be able to remove the kubeadmconstants.KubeletRunDirectory directory later
		fmt.Printf("[reset] Unmounting mounted directories in %q\n", kubeadmconstants.KubeletRunDirectory)
		// In case KubeletRunDirectory holds a symbolic link, evaluate it
		// 防止是软连接用绝对路径
		kubeletRunDir, err := absoluteKubeletRunDirectory()
		if err == nil {
			// Only clean absoluteKubeletRunDirectory if umountDirsCmd passed without error
			r.AddDirsToClean(kubeletRunDir)
		}
	} else {
		fmt.Printf("[reset] Would unmount mounted directories in %q\n", kubeadmconstants.KubeletRunDirectory)
	}

	if !r.DryRun() {
		// 删除k8s相关容器
		klog.V(1).Info("[reset] Removing Kubernetes-managed containers")
		if err := removeContainers(utilsexec.New(), r.CRISocketPath()); err != nil {
			klog.Warningf("[reset] Failed to remove containers: %v\n", err)
		}
	} else {
		fmt.Println("[reset] Would remove Kubernetes-managed containers")
	}

	// 删除dokershim目录
	// TODO: remove the dockershim directory cleanup in 1.25
	// https://github.com/kubernetes/kubeadm/issues/2626
	r.AddDirsToClean("/var/lib/dockershim", "/var/run/kubernetes", "/var/lib/cni")

	// Remove contents from the config and pki directories
	// 删除证书配置和pki文件
	if certsDir != kubeadmapiv1.DefaultCertificatesDir {
		klog.Warningf("[reset] WARNING: Cleaning a non-default certificates directory: %q\n", certsDir)
	}
	resetConfigDir(kubeadmconstants.KubernetesDir, certsDir, r.DryRun())

	if r.Cfg() != nil && features.Enabled(r.Cfg().FeatureGates, features.RootlessControlPlane) {
		if !r.DryRun() {
			klog.V(1).Infoln("[reset] Removing users and groups created for rootless control-plane")
			if err := users.RemoveUsersAndGroups(); err != nil {
				klog.Warningf("[reset] Failed to remove users and groups: %v\n", err)
			}
		} else {
			fmt.Println("[reset] Would remove users and groups created for rootless control-plane")
		}
	}

	return nil
}
```

