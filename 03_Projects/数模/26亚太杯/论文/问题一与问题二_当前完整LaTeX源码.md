% !Mode:: "TeX:UTF-8"
%%  本模板推荐以下方式编译:
%%     1. PDFLaTeX[推荐]
%%     2. xelatex [含中文推荐]

\documentclass{apmcmthesis}

\usepackage{url}
\usepackage{subcaption}
\usepackage{float}

%%%%%%%%%%%%填写相关信息%%%%%%%%%%%%%%%%%%%%%%%%%%
\tihao{B}                            %选题
\baominghao{apmcm26308576}                 %参赛编号
\begin{document}

\pagestyle{frontmatterstyle}

\begin{abstract}
% 待全文定稿后填入摘要

\keywords{Keywords1\quad  Keywords2\quad   Keywords3}
\end{abstract}



\newpage
%目录
\tableofcontents


\newpage
\pagestyle{mainmatterstyle}
\setcounter{page}{1}
\section{绪论}

\subsection{问题背景}
转炉炼钢是长流程钢铁生产的核心环节，具有生产效率高、能耗低和成本可控等优势。吹炼过程中，氧枪以超音速射流向熔池吹入高纯度高压氧气，与铁水中的碳、硅、锰、磷等杂质发生氧化反应，通过脱碳、脱磷与脱硫调控钢水成分。工业现场通常采集两类信号监测吹炼过程：一是由烟道弯头处的拉曼烟气分析仪记录的 $\text{CO}_2$ 与 $\text{CO}$ 瞬时体积流量，二是由炉口 USB4000 光谱仪采集的火焰辐射光谱。

这两类监测手段各有局限。烟气分析通用性强、维护成本低，但炉内生成的烟气需流经较长的烟道管路才能到达取样探头与分析仪，采集信号存在明显的采样-输运时滞（Sampling-Transport Delay），且易受初始碳含量计算误差的影响，单独用于动态控制时精度受限。炉口是冶炼过程中唯一可直接观测的部位，炉口火焰光谱记录的是熔池当前的实时反应状态，而烟气分析仪测得的是气体流经烟道后的滞后数据，两者在时间上刚好形成互补。

题目提供了某钢厂 10 个真实冶炼炉次的全流程实测数据（附件 1 至附件 10）。光谱仪与烟气分析仪按相同采样周期 $\Delta t = 0.5\text{ s}$ 同步记录：每个炉次的火焰光谱在 $340.535\text{ nm}$ 至 $1027.678\text{ nm}$ 范围内包含 2047 个波长点，构成约 $1400 \times 2047$ 的光强矩阵；同步记录的数据还包括累计耗氧量（单位：$\text{m}^3$）以及构成 $1400 \times 2$ 矩阵的 $\text{CO}_2$、$\text{CO}$ 瞬时体积流量（单位：$\text{m}^3/\text{s}$）。利用两类信号在时间上的互补特征建立对齐数据集，并通过数据融合模型由实时火焰光谱预测滞后的烟气流量，能够减少烟气系统取样探头的工作频次、延长分析设备寿命，为转炉终点智能控制提供数学支持。

\subsection{问题重述}
基于上述背景与附件 1 至附件 10 提供的实测数据，本文需要建立数学模型解决以下三个问题：

\begin{itemize}
    \item \textbf{问题 1}：附件 1 至附件 10 中的火焰光谱数据是按时间和波长排列的离散数据，且存在异常值。要求剔除数据中的异常点，对离散光谱数据进行连续化重构，并绘制全部 10 个炉次的三维连续光谱图。
    \item \textbf{问题 2}：附件 1 至附件 10 中的累计耗氧量与烟气数据均为含有异常值的离散时间序列。要求用数学方法剔除异常值并重构连续信号，绘制氧枪流量、累计耗氧量、$\text{CO}_2$ 浓度、$\text{CO}$ 浓度、$\text{CO}_2$ 排放总量及 $\text{CO}$ 排放总量随时间变化的连续曲线；探究累计耗氧量与 $\text{CO}_2$ 浓度、$\text{CO}$ 浓度、$\text{CO}_2$ 排放总量及 $\text{CO}$ 排放总量之间的关系；考虑到烟气信号滞后于累计耗氧量与氧枪流量的变化，且氧枪流量越高对应的烟气信号时滞越小，计算不同氧枪流量下烟气信号的滞后时间。
    \item \textbf{问题 3}：累计耗氧量与光谱数据反映了转炉熔池内物理化学反应的实时信息，而烟气数据反映了滞后信息。要求结合问题 1 和问题 2 得到的连续信号模型与时滞计算结果，研究光谱数据与烟气数据在时间轴上的实时/滞后互补关系，利用炉口火焰光谱数据实时预测烟气信息，并使用全部 10 个附件的数据集对预测模型进行测试；同时，由于不同附件数据集训练出的模型存在差异，需将这些模型进行整体融合或分组融合，使融合后的模型具备良好的泛化能力，并在全部 10 个数据集的验证中均取得良好效果。
\end{itemize}


\section{问题分析}

\subsection{问题一的分析}
问题一要求剔除 10 个炉次离散火焰光谱数据中的异常点，构建时间-波长二维连续重构模型，并绘制全部 10 个炉次的三维连续光谱图。每个炉次的原始光谱是一个行数约为 1400（时间步长 $0.5\text{ s}$）、列数为 2047（波长范围约 $340\sim 1028\text{ nm}$）的二维矩阵。

解决该问题的核心难点在于区分“单通道脉冲坏点”与“真实窄谱峰”。在转炉吹炼过程中，钾、钠等碱金属原子的受激辐射会在特定波长处形成仅跨越 $1\sim 3$ 个波道的锐利特征峰；同时，现场电磁干扰或传感器瞬态暗电流也会产生单点高值毛刺。若直接对单帧光谱施加大窗口平滑或固定阈值截断，极易将真实窄峰削平。此外，原始数据中还存在达到传感器量程上限（$65\,535$）的削顶读数以及局部采样停顿造成的记录断点。

针对上述特征，本文分三步建立重构模型：首先进行数据预处理，剔除重复波道的冗余权重，标记量程削顶位置并切分记录断点；其次，利用“真实谱峰在相邻帧具有时间持续性、或在同帧两侧具有峰肩过渡，而随机坏点在时间和波长方向均孤立”的特性，结合帧内中位数绝对偏差（MAD）自适应尺度识别并剔除孤立异常点；最后，在波长方向按实际物理波长差建立局部二次回归模型修补缺口、平滑噪声，并在二维网格面元上采用分片双线性插值构造非负连续光谱曲面。

\subsection{问题二的分析}
问题二要求对 10 个炉次的累计耗氧量和 $\text{CO}_2$、$\text{CO}$ 瞬时流量数据进行去噪与连续重构，绘制六项过程指标随时间变化的连续曲线，进而分析累计耗氧量与四项烟气指标的关系，并计算不同氧枪流量下烟气信号的滞后时间。由于原始氧表仅记录存在分次刷新台阶的累计耗氧量，直接数值差分求导会放大台阶噪声并产生负流量，本文先将氧枪瞬时流量表示为非负三次 B 样条，利用其解析积分基拟合累计耗氧序列，并对两路烟气流量采用相同样条重构与解析积分，一次性获得六条光滑连续曲线。在此基础上，将时间轴转换为累计耗氧量坐标，结合 $20\text{ s}$ 局部窗口产气比与四等分耗氧进度下的 $\text{CO}$ 体积份额，刻画不同脱碳阶段的耗氧-产气关系。最后，针对主吹期流量恒定导致全段互相关易出伪峰的问题，综合吹炼启动期的反比输运模型与运行期供氧变动事件的去趋势互相关分析，计算不同氧枪流量档位下的烟气滞后时间。


\section{模型假设}

\begin{enumerate}
    \item \textbf{量程上限截顶假设}：假设原始光谱中读数等于 $65\,535$（即 $2^{16}-1$）的数据点为强光超出传感器量程导致的削顶（右删失）观测，该读数仅代表真实辐射强度的观测下界，不作为普通有效点参与局部多项式平滑。
    \item \textbf{局部谱形平滑假设}：假设在未发生削顶的局部窄波长窗口（$5\sim 7$ 个相邻波道，跨度约 $1.5\sim 2.5\text{ nm}$）内，火焰连续热辐射背景与发射峰轮廓可由关于实际波长的低阶多项式近似。
    \item \textbf{帧间线性过渡假设}：假设在同一连续记录段内，相邻两个采样时刻（间隔 $\Delta t = 0.5\text{ s}$）之间的光谱强度呈线性平稳过渡；对于存在未记录采样停顿的断点两侧，不作跨段插值连接。
    \item \textbf{物理流量非负假设}：假设氧枪瞬时吹氧流量及 $\text{CO}_2$、$\text{CO}$ 瞬时体积流量在冶炼全过程均非负（$q(t) \ge 0,\; y_g(u) \ge 0$），原始累计耗氧序列中的短时平台与单步阶跃为流量计分次刷新所致，真实累计耗氧体积在连续记录段内单调非减。
    \item \textbf{烟道一维受迫输运假设}：假设炉内生成的混合烟气在定截面烟道管路中作一维受迫对流输运，给定吹氧强度下的有效滞后时间 $\tau$ 综合反映炉内混合、管路输运及分析仪响应的总延迟。
\end{enumerate}


\section{符号说明}

\begin{table}[htbp]
\centering
\caption{主要符号说明}
\label{tab:notations}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabularx}{\textwidth}{c X c}
\toprule
符号 & 含义说明 & 单位 \\
\midrule
$k, i, j$ & 炉次索引（$k=1,\dots,10$）、时间帧索引（$i=1,\dots,n_k$）、波道索引（$j=1,\dots,2047$） & 无量纲 \\
$t_{ki}, \lambda_{kj}$ & 第 $k$ 炉第 $i$ 帧的相对采样时间（$t_{ki} = 0.5(s_{ki} - s_{k1})$）及第 $j$ 个波道的物理波长 & $\text{s},\; \text{nm}$ \\
$X_{kij}, L_{ij}$ & 第 $k$ 炉第 $i$ 帧第 $j$ 个波道的原始辐射光强读数及局部回归后的重构节点光强 & $\text{W/cm}^2$ \\
$r_{ij}, d_i$ & 帧内五波道局部中位数残差及该帧残差的稳健尺度（MAD） & $\text{W/cm}^2$ \\
$c, w$ & 异常候选判别阈值倍数（无量纲）与局部二次拟合窗口宽度（波道数） & — \\
$S_k(t, \lambda)$ & 第 $k$ 炉在时间-波长二维连续域上的重构光谱曲面函数 & $\text{W/cm}^2$ \\
$g \in \{\text{CO}_2, \text{CO}\}$ & 烟气分析仪监测的两种碳氧化物气体通道标识 & 无量纲 \\
$s, u$ & 同一采样时钟轴上的炉口反应源时刻 $s$ 与烟气分析仪到达时刻 $u$ & $\text{s}$ \\
$V_i, \widehat{V}(t)$ & 第 $i$ 帧原始累计耗氧量读数及连续重构后的累计耗氧量曲线 & $\text{m}^3$ \\
$q^{\text{off}}(t), q^{\text{on}}(s)$ & 离线重构的氧枪瞬时流量及基于历史 $10\text{ s}$ 尾窗的在线氧枪流量 & $\text{m}^3/\text{s}$ \\
$y_{ig}, \widehat{y}_g(u)$ & 气体 $g$ 的原始瞬时体积流量读数及连续重构后的瞬时体积流量 & $\text{m}^3/\text{s}$ \\
$E_g^{\text{obs}}(u)$ & 气体 $g$ 在有效记录区间内经解析积分得到的累计排放总量 & $\text{m}^3$ \\
$B_m(t), I_m(t)$ & 结点间距为 $10\text{ s}$ 的三次 B 样条基函数及其解析积分基函数 & — \\
$\tau_g(q)$ & 氧枪流量 $q$ 下气体通道 $g$ 相对于炉口反应的滞后时间 & $\text{s}$ \\
\bottomrule
\end{tabularx}
\end{table}

\section{模型的建立与求解}

\subsection{问题一模型的建立与求解}

\subsubsection{数据预处理}
对附件 1 至附件 10 共 $13\,589$ 帧原始光谱矩阵进行核查后，在进入回归拟合前需对三类特殊数据状态进行预处理（以下固定单个炉次，省略炉次下标 $k$）：

\textbf{（1）波长网格核查与重复波道去重}

十个附件分为两个采集批组：附件 1--4（\texttt{6B03205}--\texttt{6B03271}）的波长范围为 $340.535\sim 1027.963\text{ nm}$，附件 5--10（\texttt{6C02057}--\texttt{6C02108}）的波长范围为 $339.390\sim 1023.303\text{ nm}$。同一炉次内相邻波道的波长差并非严格常数（在 $0.285\sim 0.377\text{ nm}$ 之间缓变），因此后续回归直接采用表头记录的物理波长值。此外，十个炉次中共有 26 对相邻波道在全炉所有帧上的读数完全一致。为避免重复列在最小二乘拟合和尺度估计中使局部权重翻倍，每个重复波道组仅保留一个代表列赋予权重 $\omega_\ell = 1$，其余重复列的拟合权重设为 $\omega_\ell = 0$。

\textbf{（2）量程触顶（削顶）数据标记}

在附件 2、4 以及附件 5--10 中，共检测到 $14\,805$ 个读数恰好等于 $65\,535$ 的单元，其中附件 6（\texttt{6C02058}）在 40 帧内出现 $5\,805$ 个触顶点，附件 5（\texttt{6C02057}）在 12 帧内出现 $4\,168$ 个触顶点，最长连续触顶带跨越 915 个波道。由于 $65\,535$ 是传感器 16 位采样上限，若将其视作普通异常点剔除并用两侧未触顶读数插值，会导致估计值低于已知下界（例如附件 5 第 171 帧在 $714.667, 715.002, 715.337\text{ nm}$ 处的读数依次为 $65\,187, 65\,535, 65\,341$，线性内插会给出 $65\,264 < 65\,535$）。因此，将所有等于 $65\,535$ 的节点标记为“疑似削顶”，从定量回归锚点中排除，仅在质量图层保留 $65\,535$ 作为观测下界。

\textbf{（3）采样记录断点切分}

联合检查各炉的光谱、累计耗氧量与烟气流量序列发现，附件 5 和附件 7 内部各存在一处记录断点，断点前后的原始数据跳变见表 \ref{tab:q1_breaks} 和图 \ref{fig:q1_breaks}。在单个名义步长（$0.5\text{ s}$）内，两炉的累计耗氧量分别跃增了 $54.91\text{ m}^3$ 和 $30.22\text{ m}^3$，$\text{CO}$ 流量从 $30\text{ m}^3/\text{s}$ 以上骤降至 $0.5\text{ m}^3/\text{s}$ 左右，图 \ref{fig:q1_breaks} 虚线两侧的光谱强度分布也发生了阶段性跳变。这说明相邻两帧之间存在未记录的采样中断。为此，模型在断点处将时间轴切分为两个独立连续段，后续的相邻帧核查与时间插值均在各自连续段内部独立执行。

\begin{table}[htbp]
\centering
\caption{附件 5 与附件 7 记录断点前后的多模态数据变化}
\label{tab:q1_breaks}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{lcccc}
\toprule
附件与炉号 & 相邻帧序号 & 名义时间 (s) & 累计耗氧量 ($\text{m}^3$) & $\text{CO}_2 / \text{CO}$ 流量 ($\text{m}^3/\text{s}$) \\
\midrule
附件 5 (\texttt{6C02057}) & $752 \to 753$ & $375.5 \to 376.0$ & $3393.75 \to 3448.66$ & $13.32 \to 0.63 \;/\; 30.37 \to 0.51$ \\
附件 7 (\texttt{6C02059}) & $1435 \to 1436$ & $717.0 \to 717.5$ & $3405.98 \to 3436.20$ & $12.40 \to 0.30 \;/\; 33.09 \to 0.52$ \\
\bottomrule
\end{tabular}
\end{table}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.92\textwidth]{figures/问题一/图6.png}
\caption{附件 5（面板 A）与附件 7（面板 B）记录断点前后的原始光谱分布}
\label{fig:q1_breaks}
\end{figure}

\subsubsection{异常点识别与连续曲面重构模型的建立}

\textbf{（1）基于时间与谱形支持的异常点识别}

对于第 $i$ 帧内的非削顶有效波道 $j$，取五波道滑动窗口 $\{j-2, j-1, j, j+1, j+2\}$，计算当前读数相对于局部中位数的残差 $r_{ij}$：
\begin{equation}
r_{ij} = X_{ij} - \operatorname{median}_{|h|\le 2} X_{i, j+h}
\label{eq:q1_residual}
\end{equation}
由于不同吹炼阶段的辐射总强度差异很大，采用该帧所有有效残差的中位数绝对偏差（MAD）计算帧内自适应稳健尺度 $d_i$：
\begin{equation}
d_i = \max\left\{\epsilon,\; 1.4826\,\operatorname{median}_{j}\left|r_{ij} - \operatorname{median}_{\ell} r_{i\ell}\right|\right\}
\label{eq:q1_mad}
\end{equation}
其中 $\epsilon = 1\text{ W/cm}^2$ 为防止暗背景平坦帧尺度退化的下限。当 $|r_{ij}| > c \cdot d_i$（$c$ 为阈值倍数）时，将 $(i,j)$ 列为异常候选点，并进一步核查两项邻域支持：
\begin{itemize}
    \item \textbf{时间支持}：在同一连续段的前一帧 $i-1$ 或后一帧 $i+1$ 中，若同波道或左右相邻一波道 $j' \in \{j-1, j, j+1\}$ 存在与 $r_{ij}$ 同号、且满足 $|r_{i',j'}| > 3 d_{i'}$ 的残差，则记为具备时间支持；
    \item \textbf{谱形支持}：在同一帧内以 $(\lambda_{j-2}, X_{i,j-2})$ 与 $(\lambda_{j+2}, X_{i,j+2})$ 连线作为局部基线，若紧邻波道 $j-1$ 或 $j+1$ 相对于该基线存在与 $r_{ij}$ 同号、且幅度超过 $2 d_i$ 的偏离，则说明该点两侧存在真实峰肩过渡，记为具备谱形支持。
\end{itemize}

综合两项判据，候选点 $(i,j)$ 的分类规则为：
\begin{equation}
\text{判定状态}(i,j) = 
\begin{cases} 
\text{保留（真实窄峰）}, & \text{具备时间支持或谱形支持}, \\
\text{剔除（确认孤立异常）}, & \text{两类支持均缺失且邻域观测完整}, \\
\text{保留并记待核}, & \text{位于边界或邻域含削顶、缺测}.
\end{cases}
\label{eq:q1_decision}
\end{equation}

\textbf{（2）波长方向的局部二次拟合与缺口插补}

剔除孤立异常点后，记第 $i$ 帧剩余的有效观测波道集合为 $\mathcal{V}_i$。对目标波道 $j$，取宽度为 $w = 2m+1$ 的局部窗口 $\mathcal{N}_j = \{j-m, \dots, j+m\}$，以实际物理波长差定义归一化局部坐标：
\begin{equation}
z_\ell = \frac{\lambda_\ell - \lambda_j}{H_j}, \qquad H_j = \max_{\ell \in \mathcal{N}_j} |\lambda_\ell - \lambda_j|, \quad \ell \in \mathcal{N}_j
\label{eq:q1_zcoord}
\end{equation}
在有效邻域 $\mathcal{N}_j \cap \mathcal{V}_i$ 上求解加权局部二次最小二乘回归：
\begin{equation}
(\widehat{b}_{0,ij}, \widehat{b}_{1,ij}, \widehat{b}_{2,ij}) = \arg\min_{b_0, b_1, b_2} \sum_{\ell \in \mathcal{N}_j \cap \mathcal{V}_i} \omega_\ell \left(X_{i\ell} - b_0 - b_1 z_\ell - b_2 z_\ell^2\right)^2
\label{eq:q1_poly}
\end{equation}
窗口中心 $z_j = 0$ 处的拟合值即为截距 $\widehat{b}_{0,ij}$。考虑光强的非负性，取节点重构值为：
\begin{equation}
L_{ij} = \max\left(0,\; \widehat{b}_{0,ij}\right)
\label{eq:q1_node}
\end{equation}
对于同一炉次内无缺失的完整窗口，各帧共用同一设计矩阵 $\bm{A}_j$，可预先求得伪逆截距权重向量 $\bm{w}_j^\top = (1,0,0)\bm{A}_j^+$ 并组成稀疏矩阵 $\bm{W}$，通过 $\bm{X}\bm{W}^\top$ 批量计算全炉节点值，仅在含屏蔽点的窗口局部重算权重。对于被剔除异常点形成的普通缺口（宽度不超过 5 个波道且两侧各有至少 2 个有效锚点）：单点缺口直接用左右紧邻有效锚点按实际波长作线性插值；$2\sim 5$ 个波道的缺口在窗口具备双侧支持且有效点不少于 3 个时采用式 \eqref{eq:q1_poly} 二次拟合，否则回退为双侧最近有效锚点的波长线性插值。

\textbf{（3）时间-波长二维双线性连续曲面构造}

在得到离散网格重构矩阵 $\bm{L} = (L_{ij})$ 后，对同一连续段内四角均具备定量支持的网格面元 $\Omega_{ij} = [t_i, t_{i+1}] \times [\lambda_j, \lambda_{j+1}]$，定义局部归一化坐标：
\begin{equation}
u = \frac{t - t_i}{t_{i+1} - t_i} \in [0,1], \qquad v = \frac{\lambda - \lambda_j}{\lambda_{j+1} - \lambda_j} \in [0,1]
\label{eq:q1_uv}
\end{equation}
构造面元上的分片双线性连续曲面：
\begin{equation}
S(t, \lambda) = (1-u)(1-v)L_{ij} + (1-u)v L_{i,j+1} + u(1-v)L_{i+1,j} + u v L_{i+1,j+1}
\label{eq:q1_bilinear}
\end{equation}
该曲面在相邻面元公共边上严格连续（$C^0$ 连续），且因四项权重非负、和为 1，面元内部取值恒非负且不超出四角节点值的范围。绘图抽稀时，仅当粗面片覆盖的所有底层原始面元四角均合法时才绘制曲面片，削顶无支持区与记录断点处保持留空。

\subsubsection{模型求解与结果分析}

\textbf{（1）模型参数的确定}

为客观确定阈值倍数 $c$ 与局部拟合窗口宽度 $w$，我们在每炉的起始、中段和末段分别划分出独立的\textbf{调参区}与\textbf{报告区}。在调参区上对候选组合 $c \in \{6, 8\}$、$w \in \{5, 7\}$ 进行对比测试：一方面根据原始光谱几何形态与邻帧连续性预先选出 $1\,199$ 个真实窄峰作为保护样例，检验各参数下的误删数量；另一方面通过单点与连续三波道遮蔽实验计算共同可评位置上的炉次等权 MAE 与峰面积偏差中位数，结果如表 \ref{tab:q1_tuning} 所示。

\begin{table}[htbp]
\centering
\caption{调参区上不同阈值 $c$ 与窗口宽度 $w$ 的对比结果}
\label{tab:q1_tuning}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{cccccc}
\toprule
阈值倍数 $c$ & 窗口宽度 $w$ & 可评炉数 & 调参区 MAE ($\text{W/cm}^2$) & 保护峰误删 / 总数 & 峰面积偏差中位数 \\
\midrule
$6$ & $5$ & $10$ & $101.44$ & $2 / 1199$ & $2.75$ \\
$6$ & $7$ & $10$ & $110.96$ & $2 / 1199$ & $7.07$ \\
$\bm{8}$ & $\bm{5}$ & $\bm{10}$ & $\bm{101.44}$ & $\bm{0 / 1199}$ & $\bm{2.74}$ \\
$8$ & $7$ & $10$ & $111.06$ & $0 / 1199$ & $7.04$ \\
\bottomrule
\end{tabular}
\end{table}

由表 \ref{tab:q1_tuning} 可见，当窗口宽度从 $w=5$ 增至 $w=7$ 时，调参区 MAE 由 $101.44\text{ W/cm}^2$ 上升至 $111\text{ W/cm}^2$ 左右，峰面积偏差中位数从 $2.74$ 增大至 $7.04\text{ W/cm}^2\cdot\text{nm}$，说明 7 波道窗口过宽，会将窄峰显著压低。在 $w=5$ 下，$c=6$ 会误删 2 个真实保护峰（如附件 6 第 184 帧 $918.459\text{ nm}$ 处的跨帧持续窄峰），而取 $c=8$ 时 $1\,199$ 个保护峰实现零误删，且峰面积偏差最小。因此，最终选定参数为 $c = 8,\; w = 5$。

\textbf{（2）异常点剔除与谱峰保留结果}

在 $c=8, w=5$ 下，十个炉次共识别并剔除 $16\,486$ 个孤立异常点（占总采样点的 $0.059\%$），其中 $16\,407$ 个节点通过邻域有效锚点完成定量插补；无定量支持的 $14\,884$ 个节点中，$14\,805$ 个为 $65\,535$ 削顶点。各炉次的筛查与重构统计见表 \ref{tab:q1_summary}（注：附件 1--4 波长范围均为 $340.535\sim 1027.963\text{ nm}$，附件 5--10 均为 $339.390\sim 1023.303\text{ nm}$）。

\begin{table}[htbp]
\centering
\caption{十个炉次的光谱数据规模、异常剔除与重构统计表}
\label{tab:q1_summary}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{lcccccc}
\toprule
炉次 (附件) & 采样帧数 & 跨度 (s) & 疑似削顶点 & 排除坏点 & 定量插补点 & 最大光强 ($\text{W/cm}^2$) \\
\midrule
\texttt{6B03205} (附件 1) & $1\,479$ & $739.0$ & $0$ & $550$ & $548$ & $62\,239$ \\
\texttt{6B03206} (附件 2) & $1\,311$ & $655.0$ & $13$ & $1\,481$ & $1\,479$ & $65\,081$ \\
\texttt{6B03270} (附件 3) & $1\,413$ & $706.0$ & $0$ & $1\,000$ & $996$ & $61\,531$ \\
\texttt{6B03271} (附件 4) & $1\,420$ & $709.5$ & $10$ & $1\,358$ & $1\,354$ & $65\,323$ \\
\texttt{6C02057} (附件 5) & $775$ & $387.0$ & $4\,168$ & $1\,458$ & $1\,452$ & $65\,533$ \\
\texttt{6C02058} (附件 6) & $1\,343$ & $671.0$ & $5\,805$ & $1\,458$ & $1\,446$ & $65\,579$ \\
\texttt{6C02059} (附件 7) & $1\,555$ & $777.0$ & $100$ & $2\,288$ & $2\,277$ & $65\,485$ \\
\texttt{6C02060} (附件 8) & $1\,373$ & $686.0$ & $326$ & $1\,945$ & $1\,941$ & $65\,546$ \\
\texttt{6C02107} (附件 9) & $1\,459$ & $729.0$ & $2\,110$ & $2\,420$ & $2\,402$ & $65\,533$ \\
\texttt{6C02108} (附件 10) & $1\,461$ & $730.0$ & $2\,273$ & $2\,528$ & $2\,512$ & $65\,530$ \\
\bottomrule
\end{tabular}
\end{table}

图 \ref{fig:q1_peak_excl} 展示了附件 1（\texttt{6B03205}）中保护峰保留与孤立异常点剔除的两个局部实例。在面板 A（第 468 帧，$768.930\text{ nm}$）中，中心峰值点因具备相邻帧持续响应与两侧峰肩支撑被保留，重构后峰位偏差为 $0.0000\text{ nm}$，扣底峰高保留率为 $79.03\%$；在面板 B（第 30 帧，$512.659\text{ nm}$）中，单点读数从约 $540\text{ W/cm}^2$ 突跳至 $1\,767.00\text{ W/cm}^2$（残差达 $39.63\,d_i$），且前后帧与左右波道均无支撑，模型将其剔除并插补修复为 $541.12\text{ W/cm}^2$。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.92\textwidth]{figures/问题一/图3.png}
\caption{保护峰保留（面板 A）与孤立异常剔除（面板 B）的局部重构实例}
\label{fig:q1_peak_excl}
\end{figure}

对报告区全部未削顶原始谱峰按半高宽（FWHM）分层统计（表 \ref{tab:q1_fwhm}）：半高宽超过 2 个波道的较宽谱峰匹配率达 $90.69\%$，峰高保留率中位数为 $93.13\%$，峰位偏差中位数为 $0.0000\text{ nm}$；半高宽不超过 2 个波道的极窄峰匹配率为 $34.91\%$，匹配窄峰的峰高保留率中位数为 $67.12\%$。

\begin{table}[htbp]
\centering
\caption{报告区按原始谱峰半高宽分层的峰形统计结果}
\label{tab:q1_fwhm}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{lccccc}
\toprule
谱峰半高宽分层 & 匹配数 / 原峰数 & 匹配率 & 峰高保留率 & 峰位偏差 (nm) & 峰面积偏差 \\
\midrule
较宽谱峰（$\text{FWHM} > 2$ 波道） & $22\,759 / 25\,095$ & $90.69\%$ & $93.13\%$ & $0.0000$ & $3.13$ \\
极窄谱峰（$\text{FWHM} \le 2$ 波道） & $19\,731 / 56\,512$ & $34.91\%$ & $67.12\%$ & $0.0000$ & $2.65$ \\
\bottomrule
\end{tabular}
\end{table}

\textbf{（3）十个炉次的三维连续光谱图与演化特征}

图 \ref{fig:q1_all_surfaces} 给出了重构后的全部十个炉次三维连续火焰光谱图（各炉光强按本炉排除削顶峰区后的最大有效节点光强进行归一化，色条范围 $0\sim 1$，无定量支持与宽削顶区留空）。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.98\textwidth]{figures/问题一/图2.png}
\caption{十个炉次的三维连续火焰光谱图}
\label{fig:q1_all_surfaces}
\end{figure}

由图 \ref{fig:q1_all_surfaces} 和表 \ref{tab:q1_summary} 可归纳出十个炉次火焰光谱的三项分布与演化规律：
\begin{enumerate}
    \item \textbf{吹炼全周期阶段性演化}：吹炼前期（约前 $100\sim 150\text{ s}$），全波段光强维持在 $2\,500\text{ W/cm}^2$ 以下的低水平平坦区；进入主脱碳期后，炉内剧烈脱碳生成大量高温 $\text{CO}$ 并在炉口燃烧，连续热辐射与特征发射峰迅速升高，形成多峰脉动的强辐射带；吹炼末期随着碳含量降低，火焰辐射强度快速回落。
    \item \textbf{主辐射波段与特征峰位置}：辐射能量主要分布在可见光红光至近红外波段（$580\sim 900\text{ nm}$），短波段（$340\sim 480\text{ nm}$）全程强度较低。在 $765\sim 770\text{ nm}$（钾原子特征双线）和 $589\text{ nm}$ 附近（钠原子特征线）存在贯穿吹炼中后期的窄带高脊线，其中 \texttt{6B} 批组四炉的最大定量节点均位于 $765.297\sim 768.270\text{ nm}$ 范围内。
    \item \textbf{批次间辐射强度分布差异}：\texttt{6B} 批组（附件 1--4）在中前期强度爬升较缓，峰值集中在吹炼后半段（$550\sim 715\text{ s}$），几乎无大面积量程削顶；\texttt{6C} 批组（附件 5--10）在吹炼中段（$100\sim 550\text{ s}$）即维持高强度宽谱带辐射，附件 5、6、9、10 在 $600\sim 900\text{ nm}$ 区间出现大量触顶饱和点。这一批组特征差异为问题三按炉次分组融合预测提供了依据。
\end{enumerate}

\subsubsection{模型检验与敏感性分析}

\textbf{（1）多模式遮蔽重构检验与人工尖点注入测试}

在独立报告区中，分别开展单波道遮蔽、连续三波道遮蔽和整帧时间遮蔽实验，并在有效非峰区独立注入幅度为 $3\,d_i, 6\,d_i, 10\,d_i$ 的单点人工尖点（每组 300 次试验），结果见图 \ref{fig:q1_validation} 和表 \ref{tab:q1_spike}。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.95\textwidth]{figures/问题一/图5.png}
\caption{报告区逐炉遮蔽重构误差（面板 A：单点；面板 B：三波道；面板 C：整帧）与人工尖点检出率（面板 D）}
\label{fig:q1_validation}
\end{figure}

\begin{table}[htbp]
\centering
\caption{报告区人工尖点注入与检出结果统计}
\label{tab:q1_spike}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{ccccccc}
\toprule
注入幅度 & 试验次数 & 命中数 & 检出率 & 重算未越阈 & 获邻域支持 & 附带误删 / 可筛查点数 \\
\midrule
$3\,d_i$ & $300$ & $0$ & $0.00\%$ & $298$ & $2$ & $2 / 1\,830\,359$ \\
$6\,d_i$ & $300$ & $3$ & $1.00\%$ & $282$ & $15$ & $3 / 1\,830\,834$ \\
$10\,d_i$ & $300$ & $146$ & $48.67\%$ & $59$ & $95$ & $6 / 1\,826\,446$ \\
\bottomrule
\end{tabular}
\end{table}

波长方向上，单点与三波道遮蔽的十炉等权平均 MAE 分别为 $100.21\text{ W/cm}^2$ 和 $116.10\text{ W/cm}^2$（占满量程约 $0.15\%\sim 0.18\%$），连续五波道缺口在十炉上均达到 $100\%$ 覆盖率。时间方向上（图 \ref{fig:q1_validation}C），整帧遮蔽双侧插值重构的平均 MAE 为 $2\,247.11\text{ W/cm}^2$，较“上一帧保持”（$2\,823.88\text{ W/cm}^2$）降低了 $20.42\%$，在 9/10 个炉次上误差更低；同时在 9/10 个炉次上优于直接对原始强度作双侧时间线性插值（$2\,247.37\text{ W/cm}^2$）。在尖点注入测试中（表 \ref{tab:q1_spike}），$10\,d_i$ 强毛刺的检出率为 $48.67\%$，且在超过 182 万个背景可筛查点中的附带排除仅 6 例。

\textbf{（2）峰顶平滑强度的敏感性分析}

为考察减弱峰顶平滑对窄峰还原与抗噪能力的影响，设置“窄峰保护分支”作为对照（即对半高宽不超过 2 个波道、具备邻域支持的局部峰顶，当二次回归值低于原始读数时，取两者等权平均值作为输出）。两种方案在报告区上的成对比较如图 \ref{fig:q1_tradeoff} 所示。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.92\textwidth]{figures/问题一/图7.png}
\caption{主方案与窄峰保护分支的谱峰匹配率（面板 A）及峰顶尖点扰动传递率（面板 B）对比}
\label{fig:q1_tradeoff}
\end{figure}

如图 \ref{fig:q1_tradeoff}A 所示，窄峰保护分支将报告区极窄峰（$\le 2$ 波道）的匹配率从 $34.91\%$ 提高到 $39.64\%$；但如图 \ref{fig:q1_tradeoff}B 所示，当在有支撑峰顶叠加 $3\,d_i, 6\,d_i, 10\,d_i$ 的单点尖点扰动时，主方案依靠五点二次回归将平均扰动传递率控制在 $48.57\%$，而窄峰保护分支的扰动传递率则上升至 $73.89\%$、$71.58\%$ 和 $68.53\%$。综合考虑曲面整体的平滑性与抗噪稳健性，本文最终采用固定五点局部二次拟合的主方案。

\subsection{问题二模型的建立与求解}

\subsubsection{数据预处理}
对十个炉次的第 2049 列（累计耗氧量）及第 2050、2051 列（$\text{CO}_2$、$\text{CO}$ 瞬时体积流量，单位均为 $\text{m}^3/\text{s}$）按以下三步进行预处理：

\textbf{（1）记录断点切分}

与问题一保持一致，附件 5 在序号 $752 \to 753$（名义时间 $375.5 \to 376.0\text{ s}$）、附件 7 在序号 $1435 \to 1436$（名义时间 $717.0 \to 717.5\text{ s}$）处存在记录中断，断点前后的累计耗氧量跨界差额分别为 $54.91\text{ m}^3$ 和 $30.22\text{ m}^3$。模型在断点处将序列切分为独立连续段，各段以段首表值为起点分别拟合，跨界差额单独记账，不平摊为断点处的虚假尖峰流量。

\textbf{（2）累计氧表异常值处理}

原始累计耗氧量序列中的短时恒值平台与正阶跃反映了流量计的分次累加刷新，予以完整保留；仅对违背体积守恒的单行回跳或非有限缺测值进行屏蔽。

\textbf{（3）烟气尾段停测截断}

吹炼起始阶段的零读数作为有效观测保留，孤立尖点由后续 Huber 稳健损失函数抑制。针对附件 5 和附件 7 在记录断点后烟气读数骤降至近零的现象，设定统一的尾段终止判据：取候选断点前 $60\text{ s}$ 有效读数的中位数 $m$（$m \ge 1\text{ m}^3/\text{s}$），若断点后首点低于 $0.1m$、随后连续 $2\text{ s}$ 低于 $0.2m$ 且直至末行至少 $80\%$ 的读数低于 $0.2m$，则判定烟气有效采集终止，将该低值尾段从积分与时滞配对域中截断（附件 5 烟气有效覆盖时长为 $375.5\text{ s}$，附件 7 为 $717.0\text{ s}$，其余八炉均为全跨度覆盖）。

\subsubsection{过程信号连续重构模型}

\textbf{（1）氧枪瞬时流量反演}

在连续记录段 $[t_0, t_E]$ 内，累计耗氧量 $V(t)$ 是氧枪瞬时流量 $q^{\text{off}}(t)$ 的时间积分。以固定间距 $10\text{ s}$ 布置三次 B 样条基函数 $\{B_m(t)\}_{m=1}^M$，将未知的氧枪流量展开为非负样条组合，并直接写出累计耗氧量的解析积分表达式：
\begin{equation}
q^{\text{off}}(t) = \sum_{m=1}^{M} c_m B_m(t), \quad c_m \ge 0, \qquad \widehat{V}(t) = V(t_0) + \sum_{m=1}^{M} c_m I_m(t)
\label{eq:q2_ox_spline}
\end{equation}
其中 $I_m(t) = \int_{t_0}^{t} B_m(v)\,\mathrm{d}v$ 为三次 B 样条基的解析原函数。由于 $B_m(t) \ge 0$ 且 $I_m(t)$ 单调非减，约束 $c_m \ge 0$ 能严格保证氧枪瞬时流量非负且累计耗氧曲线单调非减。

样条系数 $\bm{c} = (c_1, \dots, c_M)^\top$ 通过最小化带二阶差分惩罚的 Huber 稳健目标函数求解：
\begin{equation}
\min_{\bm{c} \ge \bm{0}} \;\; \frac{1}{n} \sum_{i \in \mathcal{V}_V} \rho_{\text{Huber}}\!\left(\frac{V_i - \widehat{V}(t_i)}{s_V}\right) + \lambda_q \frac{\|\bm{D}^2 \bm{c}\|_2^2}{\max(1, M-2)\, s_q^2}
\label{eq:q2_ox_obj}
\end{equation}
式中，$\mathcal{V}_V$ 为段内有效观测集；$\rho_{\text{Huber}}(z)$ 在 $|z|\le 1$ 时取 $\frac{1}{2}z^2$、在 $|z|>1$ 时取 $|z|-\frac{1}{2}$；$\bm{D}^2$ 为系数二阶差分矩阵；$s_V$ 与 $s_q$ 为无量纲化尺度因子；$\lambda_q$ 为平滑参数（在调参区通过 $5\text{ s}$ 连续块留出验证从 $\{0.01, 0.1, 1.0\}$ 中选定为 $\lambda_q = 0.1$）。此外，为满足在线时滞触发与第三问实时预测的因果性要求，同步定义过去 $h = 10\text{ s}$ 尾窗上的在线氧枪流量 $q^{\text{on}}(s)$：
\begin{equation}
(\widehat{b}_0, \widehat{b}_1) = \arg\min_{b_0, b_1} \sum_{t_i \in [s-h, s]} \rho_{\text{Huber}}\!\left(\frac{V_i - b_0 - b_1(t_i - s)}{s_V^{\text{train}}}\right), \qquad q^{\text{on}}(s) = \max(0, \widehat{b}_1)
\label{eq:q2_qon}
\end{equation}

\textbf{（2）烟气流量重构与累计排放积分}

对分析仪时钟 $u$ 上的 $\text{CO}_2$ 与 $\text{CO}$ 瞬时流量 $y_{ig}$（$g \in \{\text{CO}_2, \text{CO}\}$），在有效段 $r = [l_r, h_r]$ 内同样采用结点间距为 $10\text{ s}$、平滑参数 $\lambda = 0.1$ 的非负三次 B 样条重构，并由样条原函数解析积分得到累计排放总量 $E_g^{\text{obs}}(u)$：
\begin{equation}
\widehat{y}_g(u) = \sum_{m=1}^{M_r} b_{gm} B_m(u), \quad b_{gm} \ge 0, \qquad E_g^{\text{obs}}(u) = \sum_{h_j < l_r} \int_{l_j}^{h_j} \widehat{y}_g(v)\,\mathrm{d}v + \int_{l_r}^{u} \widehat{y}_g(v)\,\mathrm{d}v
\label{eq:q2_gas_spline}
\end{equation}

\textbf{（3）六条连续过程曲线求解结果}

表 \ref{tab:q2_summary} 汇总了十个炉次的有效覆盖时长、累计耗氧对账与两类气体累计排放总量（注：附件 5 与附件 7 的断点跨界氧量差额分别为 $54.91\text{ m}^3$ 与 $30.22\text{ m}^3$，其余八炉无断点差额且有效覆盖时长等于记录总跨度）；图 \ref{fig:q2_ox_curves}、图 \ref{fig:q2_gas_flows} 和图 \ref{fig:q2_gas_cum} 展示了全部十个炉次重构后的六条连续过程曲线。

\begin{table}[htbp]
\centering
\caption{十个炉次的有效覆盖时长、累计耗氧量与烟气累计排放总量统计表}
\label{tab:q2_summary}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{lcccccc}
\toprule
炉次 (附件) & 总跨度 (s) & 烟气覆盖 (s) & 氧表总差 ($\text{m}^3$) & 段内净增 & $\text{CO}_2$ 累计 ($\text{m}^3$) & $\text{CO}$ 累计 ($\text{m}^3$) \\
\midrule
\texttt{6B03205} (附件 1) & $739.0$ & $739.0$ & $3\,391.33$ & $3\,391.33$ & $9\,345.0$ & $24\,869.8$ \\
\texttt{6B03206} (附件 2) & $655.0$ & $655.0$ & $3\,319.06$ & $3\,319.06$ & $9\,633.1$ & $16\,982.1$ \\
\texttt{6B03270} (附件 3) & $706.0$ & $706.0$ & $3\,328.96$ & $3\,328.96$ & $10\,032.5$ & $23\,389.4$ \\
\texttt{6B03271} (附件 4) & $709.5$ & $709.5$ & $3\,424.90$ & $3\,424.90$ & $10\,538.4$ & $18\,601.1$ \\
\texttt{6C02057} (附件 5) & $387.0$ & $375.5$ & $3\,536.60$ & $3\,481.69$ & $4\,828.8$ & $11\,016.9$ \\
\texttt{6C02058} (附件 6) & $671.0$ & $671.0$ & $3\,339.14$ & $3\,339.14$ & $8\,384.3$ & $20\,446.3$ \\
\texttt{6C02059} (附件 7) & $777.0$ & $717.0$ & $3\,601.39$ & $3\,571.17$ & $9\,839.1$ & $20\,205.3$ \\
\texttt{6C02060} (附件 8) & $686.0$ & $686.0$ & $3\,306.63$ & $3\,306.63$ & $9\,035.5$ & $21\,374.1$ \\
\texttt{6C02107} (附件 9) & $729.0$ & $729.0$ & $3\,405.00$ & $3\,405.00$ & $10\,989.4$ & $21\,410.9$ \\
\texttt{6C02108} (附件 10) & $730.0$ & $730.0$ & $3\,487.91$ & $3\,487.91$ & $10\,353.8$ & $19\,307.3$ \\
\bottomrule
\end{tabular}
\end{table}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.95\textwidth]{figures/问题二/图1.png}
\caption{十个炉次的氧枪瞬时流量（面板 A）与累计耗氧量（面板 B）连续曲线}
\label{fig:q2_ox_curves}
\end{figure}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.95\textwidth]{figures/问题二/图2.png}
\caption{十个炉次的 $\text{CO}_2$（面板 A）与 $\text{CO}$（面板 B）瞬时体积流量连续曲线}
\label{fig:q2_gas_flows}
\end{figure}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.95\textwidth]{figures/问题二/图3.png}
\caption{十个炉次的 $\text{CO}_2$（面板 A）与 $\text{CO}$（面板 B）累计排放总量连续曲线}
\label{fig:q2_gas_cum}
\end{figure}

由表 \ref{tab:q2_summary} 和图 \ref{fig:q2_ox_curves}--图 \ref{fig:q2_gas_cum} 可以看出：
\begin{enumerate}
    \item \textbf{吹氧曲线特征}：十个炉次的总耗氧量稳定在 $3\,306.63\sim 3\,601.39\text{ m}^3$ 之间。除附件 5 外，其余九炉在前 $80\text{ s}$ 维持 $2.3\sim 2.6\text{ m}^3/\text{s}$ 的低流量成渣台阶，$80\sim 100\text{ s}$ 迅速拉升至 $6\sim 9\text{ m}^3/\text{s}$，随后在 $100\sim 700\text{ s}$ 主吹期内稳定在 $4.5\sim 5.8\text{ m}^3/\text{s}$ 的恒流平台，累计耗氧曲线（图 \ref{fig:q2_ox_curves}B）呈近似直线。附件 5（深红线）在 $387.0\text{ s}$ 名义跨度内耗氧量同样达 $3\,536.60\text{ m}^3$，故其反推流量在 $6\sim 14\text{ m}^3/\text{s}$ 之间波动，累计曲线斜率约为其余九炉的两倍。
    \item \textbf{烟气释放曲线特征}：$\text{CO}_2$ 与 $\text{CO}$ 瞬时流量呈现明显的先后错位（图 \ref{fig:q2_gas_flows}）。$\text{CO}_2$ 在吹炼 $40\sim 60\text{ s}$ 即开始爬升，于 $150\sim 250\text{ s}$ 达到峰值（$20\sim 24.48\text{ m}^3/\text{s}$），中后期缓降至 $10\sim 16\text{ m}^3/\text{s}$；$\text{CO}$ 在前 $100\text{ s}$ 接近零，$100\sim 150\text{ s}$ 后陡峭上升，在 $300\sim 650\text{ s}$ 主脱碳期维持 $35\sim 63.60\text{ m}^3/\text{s}$ 的高位平台。正常全跨度炉次的 $\text{CO}_2$ 累计排放量为 $8\,384.3\sim 10\,989.4\text{ m}^3$，$\text{CO}$ 累计排放量达 $16\,982.1\sim 24\,869.8\text{ m}^3$（图 \ref{fig:q2_gas_cum}）。
\end{enumerate}

\subsubsection{累计耗氧量对烟气排放的影响分析}

\textbf{（1）局部产气比与四阶段排放构成定义}

为剥离时间累积趋势、定量分析累计耗氧量对四项烟气指标的影响，定义两项分析指标：
\begin{enumerate}
    \item \textbf{局部窗口耗氧产气比}：在共同有效段内划分互不重叠的 $20\text{ s}$ 窗口 $[a, b]$，计算局部单位耗氧产气增量比 $R_g[a,b]$（单位：$\text{m}^3\text{ 烟气}/\text{m}^3\text{ O}_2$），并建立控制时间进度 $p \in [0,1]$ 后的回归方程：
    \begin{equation}
    R_g[a,b] = \frac{E_g^{\text{obs}}(b) - E_g^{\text{obs}}(a)}{\widehat{V}(b) - \widehat{V}(a)}, \qquad \bar{y}_{g,w} = \alpha_{k} + \gamma_1 p_w + \gamma_2 p_w^2 + \beta_g \bar{q}_{w} + \varepsilon_{w}
    \label{eq:q2_ratio_reg}
    \end{equation}
    其中 $\bar{y}_{g,w}$ 与 $\bar{q}_w$ 分别为窗口平均烟气流量与平均氧流量，$\alpha_k$ 为炉次截距。
    \item \textbf{四阶段 $\text{CO}$ 排放体积份额}：将每炉共同有效区间内的净耗氧总量四等分为四个进度阶段 $S_j \in \{[0,25\%), [25\%,50\%), [50\%,75\%), [75\%,100\%]\}$（$j=1,2,3,4$），积分计算各阶段内两气体的排放体积 $G_{k,j,\text{CO}}$ 与 $G_{k,j,\text{CO}_2}$，计算阶段 $\text{CO}$ 体积份额及十炉均值：
    \begin{equation}
    s_{k,j} = \frac{G_{k,j,\text{CO}}}{G_{k,j,\text{CO}} + G_{k,j,\text{CO}_2}}, \qquad \bar{s}_j = \frac{1}{n_j} \sum_{k=1}^{n_j} s_{k,j}
    \label{eq:q2_co_share}
    \end{equation}
\end{enumerate}

\textbf{（2）耗氧-产气关系求解结果}

以重构累计耗氧量 $\widehat{V}(u)$ 为横轴，图 \ref{fig:q2_ox_vs_flow} 与图 \ref{fig:q2_ox_vs_cum} 分别展示了十个炉次累计耗氧量与两气体瞬时流量及累计排放总量的关系曲线；图 \ref{fig:q2_co_share} 展示了四等分耗氧进度下的 $\text{CO}$ 体积份额变化，表 \ref{tab:q2_coupling} 给出了 $20\text{ s}$ 局部窗口产气比与进度校正回归结果。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.95\textwidth]{figures/问题二/图4.png}
\caption{累计耗氧量与 $\text{CO}_2$（面板 A）、$\text{CO}$（面板 B）瞬时体积流量的关系曲线}
\label{fig:q2_ox_vs_flow}
\end{figure}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.95\textwidth]{figures/问题二/图5.png}
\caption{累计耗氧量与 $\text{CO}_2$（面板 A）、$\text{CO}$（面板 B）累计排放总量的关系曲线}
\label{fig:q2_ox_vs_cum}
\end{figure}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.82\textwidth]{figures/问题二/图15.png}
\caption{按累计耗氧进度四阶段划分的 $\text{CO}$ 排放体积份额}
\label{fig:q2_co_share}
\end{figure}

\begin{table}[htbp]
\centering
\caption{$20\text{ s}$ 局部窗口耗氧产气比与进度校正回归结果}
\label{tab:q2_coupling}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{cccccc}
\toprule
气体通道 & $20\text{ s}$ 窗数 & 全段 $\Delta E / \Delta V$ 范围 & 局部 $\Delta E / \Delta V$ 中位 & 校正系数 $\beta_g$ & 进度 / 加氧后 RMSE \\
\midrule
$\text{CO}_2$ & $330$ & $1.423\sim 3.226$ & $2.787$ & $+1.015$ & $4.132 \;/\; 4.006$ \\
$\text{CO}$ & $330$ & $3.247\sim 7.332$ & $7.038$ & $-0.338$ & $7.044 \;/\; 7.036$ \\
\bottomrule
\end{tabular}
\end{table}

结合图 \ref{fig:q2_ox_vs_flow}--图 \ref{fig:q2_co_share} 和表 \ref{tab:q2_coupling} 可得出以下三点结论：
\begin{enumerate}
    \item \textbf{跨炉次演化轨迹在耗氧坐标下高度收敛}：在时间坐标（图 \ref{fig:q2_gas_flows}）中明显偏离的附件 5（深红线），转换到累计耗氧量坐标（图 \ref{fig:q2_ox_vs_flow}）后与其余九炉的峰值和拐点位置高度重合——累计耗氧达 $200\sim 400\text{ m}^3$ 时 $\text{CO}_2$ 陡升，在 $600\sim 1\,000\text{ m}^3$ 处达峰；$\text{CO}$ 则在累计耗氧跨过 $800\sim 1\,200\text{ m}^3$ 后快速爆发，在 $1\,500\sim 3\,000\text{ m}^3$ 维持高位脱碳平台，超过 $3\,100\text{ m}^3$ 后回落。这说明累计耗氧量直接决定了熔池冶金反应的推进阶段。
    \item \textbf{四阶段碳氧反应产物的主导权转换}：由图 \ref{fig:q2_co_share} 可知，当累计净耗氧进度从 $0\sim 25\%$ 进入 $25\sim 50\%$ 时，十炉平均 $\text{CO}$ 排放体积份额从 $22.8\%$ 跃升至 $64.4\%$（净增 $41.6$ 个百分点），并在后两个阶段继续升至 $75.6\%$ 和 $77.5\%$（剔除附件 5 后依次为 $21.6\%, 63.4\%, 75.5\%, 77.7\%$）。这是因为前 $25\%$ 耗氧期主要发生硅、锰氧化，少量 $\text{CO}$ 在炉内富氧条件下被二次燃烧转化为 $\text{CO}_2$；进入中后期高温主脱碳阶段后，剧烈的碳氧反应（$\text{C}+\text{O} \to \text{CO}\uparrow$）占主导地位，排气构成转为以 $\text{CO}$ 为主。
    \item \textbf{累计排放增长斜率}：在累计耗氧超过 $1\,000\text{ m}^3$ 后，累计排放量随累计耗氧量近似线性增长（图 \ref{fig:q2_ox_vs_cum}），$\text{CO}_2$ 和 $\text{CO}$ 的 $20\text{ s}$ 局部产气比中位数分别为 $2.787$ 和 $7.038\text{ m}^3/\text{m}^3$（表 \ref{tab:q2_coupling}）。通过式 \eqref{eq:q2_ratio_reg} 扣除二次时间进度趋势后，加入氧流量项仅使 $\text{CO}_2$ 和 $\text{CO}$ 的残差 RMSE 从 $4.132, 7.044\text{ m}^3/\text{s}$ 微降至 $4.006, 7.036\text{ m}^3/\text{s}$，说明主吹恒流期内累计排放与累计耗氧的线性趋势主要由反应进程累积驱动。
\end{enumerate}

\subsubsection{烟气信号的动态时滞计算}

\textbf{（1）时滞辨识方法}

分别从吹炼启动期和主吹运行期计算不同氧枪流量下的烟气滞后时间 $\tau_g(q)$：
\begin{enumerate}
    \item \textbf{启动期反比输运模型}：吹炼启动阶段气体在定容积烟道内流速与吹氧流量成正比，滞后时间满足反比关系：
    \begin{equation}
    \tau(q) = \tau_0 + \frac{\kappa}{q}
    \label{eq:q2_inv_lag}
    \end{equation}
    提取首个连续段内供氧流量首次持续 $5\text{ s}$ 越阈与两气体总流量首次持续 $5\text{ s}$ 越阈（阈值 $2\text{ m}^3/\text{s}$）的时间差作为启动实测时滞，结合起始 $10\text{ s}$ 平均氧流量拟合式 \eqref{eq:q2_inv_lag}。
    \item \textbf{运行期供氧变动事件互相关辨识}：在运行期，计算前后相邻 $10\text{ s}$ 窗口在线氧流量均值差 $|\Delta \bar{q}^{\text{on}}(s)|$，以训练炉等权第 $90$ 百分位数为阈值提取局部极大值作为供氧变动事件 $t_e$（$30\text{ s}$ 内去重），并按事件后稳定氧流量的三分位数划分为\textbf{低流量档（$1.96\sim 4.70\text{ m}^3/\text{s}$）、中流量档（$4.70\sim 5.15\text{ m}^3/\text{s}$）和高流量档（$5.15\sim 14.80\text{ m}^3/\text{s}$）}。
    
    在源窗口 $W_e = [t_e - 20\text{ s}, t_e + 20\text{ s}]$ 内，对氧流量序列（水平值或 $2\text{ s}$ 后向差分变化量）与平移窗口 $W_e + \ell$（$\ell \in [0, 60]\text{ s}$，步长 $0.5\text{ s}$）内的烟气序列分别扣除局部线性趋势，计算互相关剖面与峰值位置：
    \begin{equation}
    C_{eg}(\ell) = \operatorname{corr}\!\left(q^{\text{on}}_{e,\text{detrend}}(s),\; y_{eg,\text{detrend}}(s + \ell)\right), \qquad \widehat{\tau}_{eg} = \arg\max_{\ell \in [0,60]} |C_{eg}(\ell)|
    \label{eq:q2_xcorr}
    \end{equation}
    为剔除平稳段随机波动造成的伪峰，将同一源窗口氧流量与同炉同段内互不重叠的异位烟气窗口作错配互相关，取错配峰值绝对值的炉次等权第 $95$ 百分位数作为显著性阈值（$\text{CO}_2$ 为 $0.872$，$\text{CO}$ 为 $0.902$），仅保留峰值严格越阈且非边界宽峰的事件。
\end{enumerate}

\textbf{（2）不同氧枪流量下的时滞计算结果}

图 \ref{fig:q2_lag_profile} 展示了运行期两个代表性供氧变动事件的滞后互相关剖面，表 \ref{tab:q2_lag_summary} 汇总了不同氧枪流量区间下的时滞计算结果。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.88\textwidth]{figures/问题二/图8.png}
\caption{代表性供氧变动事件的去趋势滞后互相关剖面（蓝线：附件 10 序号 168 的 $\text{CO}$ 响应；红线：附件 6 序号 617 的 $\text{CO}_2$ 响应）}
\label{fig:q2_lag_profile}
\end{figure}

\begin{table}[htbp]
\centering
\caption{不同氧枪流量区间下的烟气信号滞后时间计算结果}
\label{tab:q2_lag_summary}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{lccccc}
\toprule
辨识阶段与信号表征 & 通道 & 氧枪流量区间 ($\text{m}^3/\text{s}$) & 候选 & 过阈数 & 滞后时间 $\widehat{\tau}_g$ (s) \\
\midrule
启动期首次持续越阈 & $\text{CO}_2+\text{CO}$ & 低启动流量 ($2.3\sim 2.6$) & $9$ 炉 & $8$ 炉 & $\bm{32.0 \sim 41.5}$ \\
启动期首次持续越阈 & $\text{CO}_2+\text{CO}$ & 高启动流量 ($7.5\sim 11.2$) & $1$ 炉 & $1$ 炉 & $\bm{10.5 \sim 14.0}$ \\
\midrule
运行期事件（水平序列） & $\text{CO}$ & 低/中流量档 ($1.96\sim 5.15$) & $9$ & $0$ & 未过阈 ($<0.902$) \\
运行期事件（水平序列） & $\text{CO}$ & 高流量档 ($5.15\sim 14.80$) & $24$ & $1$ & $\bm{24.50}$ \\
\midrule
运行期事件（$2\text{ s}$ 差分） & $\text{CO}_2$ & 低流量档 ($1.96\sim 4.70$) & $3$ & $1$ & $\bm{39.00}$ \\
运行期事件（$2\text{ s}$ 差分） & $\text{CO}_2$ & 中流量档 ($4.70\sim 5.15$) & $6$ & $1$ & $\bm{39.00}$ \\
运行期事件（$2\text{ s}$ 差分） & $\text{CO}$ & 低流量档 ($1.96\sim 4.70$) & $3$ & $1$ & $\bm{39.00}$ \\
运行期事件（$2\text{ s}$ 差分） & $\text{CO}$ & 中/高流量档 ($4.70\sim 14.80$) & $30$ & $3$ & $\bm{10.00 \sim 31.25}$ \\
\midrule
长窗口全过程（$150\text{ s}$ 窗） & $\text{CO}_2$ & 全程主吹工况 ($4.5\sim 6.5$) & $158$ 窗 & $9$ 窗 ($6$ 炉) & $\bm{36.50}$ \\
\bottomrule
\end{tabular}
\end{table}

由图 \ref{fig:q2_lag_profile} 和表 \ref{tab:q2_lag_summary} 可以得出：
\begin{enumerate}
    \item \textbf{滞后时间随氧枪流量增大而减小}：在低流量档（$1.96\sim 4.70\text{ m}^3/\text{s}$）及常规主吹中流量档（$4.70\sim 5.15\text{ m}^3/\text{s}$）下，烟道内气体流速较缓，$\text{CO}_2$ 与 $\text{CO}$ 的事件辨识时滞均在 $39.00\text{ s}$ 左右（附件 4 低流量爬坡段与附件 6 主吹段均为 $39.00\text{ s}$，与 $150\text{ s}$ 长窗口跨炉中位时滞 $36.50\text{ s}$ 及九个常规炉次启动期时滞 $32.0\sim 41.5\text{ s}$ 一致）。当氧枪流量升至高流量档（$5.15\sim 14.80\text{ m}^3/\text{s}$）时，炉内剧烈产气加快了管路输运速度：如图 \ref{fig:q2_lag_profile} 中附件 10 序号 168 事件（$q = 7.08\text{ m}^3/\text{s}$），$\text{CO}$ 互相关剖面在 $\widehat{\tau}_{\text{CO}} = 24.50\text{ s}$ 处取得显著单峰（差分序列下高流量档等权中位时滞降至 $31.25\text{ s}$，其中附件 10 为 $24.0\text{ s}$）；在启动期更高流量（附件 5，$7.5\sim 11.2\text{ m}^3/\text{s}$）下，时滞进一步缩短至 $10.5\sim 14.0\text{ s}$（全十炉下反比模型式 \eqref{eq:q2_inv_lag} 的留炉 RMSE 为 $2.74\text{ s}$，显著优于常时滞基准的 $10.94\text{ s}$）。
    \item \textbf{差分变化量对运行期事件的检出增益}：主吹期氧流量波动较小，直接用去趋势水平序列作互相关时，多数事件峰值接近但略低于负对照阈值（如图 \ref{fig:q2_lag_profile} 红线所示的附件 6 序号 617 事件，$\text{CO}_2$ 峰值位于 $39.5\text{ s}$、相关系数 $0.862$，略低于 $0.872$ 门槛）。改用 $2\text{ s}$ 后向差分序列后，低频漂移被滤除，十炉正时滞通过事件由 1 个增至 6 个（留炉外折由 1 个增至 9 个），完整呈现了从低流量档 $39.00\text{ s}$ 到高流量档 $24.0\sim 31.25\text{ s}$ 的递减趋势。
\end{enumerate}

\subsubsection{模型检验}

\textbf{（1）基于光谱预测误差的时滞互证}

为检验上述辨识时滞（高流量事件约 $24.5\text{ s}$、常规主吹约 $36.5\sim 39.0\text{ s}$）在双模态对齐中的有效性，采用留一炉交叉验证（LOHO），以源时刻 $s$ 的公共波段光谱亮度中位数及过去 $3\text{ s}$ 变化率预测时刻 $u = s + \ell$（$\ell \in [0, 60]\text{ s}$）的烟气流量。不同提前量 $\ell$ 相对于零时滞基准（$\ell = 0$）的留炉 MAE 变化率曲线见图 \ref{fig:q2_optical_lag}，四种典型对齐时钟在同一批原始分析仪观测点（$\text{CO}_2$ 共 $12\,620$ 点，$\text{CO}$ 共 $12\,626$ 点）上的留炉 MAE 对照见图 \ref{fig:q2_clock_bars} 与表 \ref{tab:q2_clock_comp}。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.88\textwidth]{figures/问题二/图9.png}
\caption{不同预测提前量下光谱诊断模型相对零时滞基准的留出炉 MAE 变化率}
\label{fig:q2_optical_lag}
\end{figure}

\begin{figure}[htbp]
\centering
\includegraphics[width=0.92\textwidth]{figures/问题二/图16.png}
\caption{四种时滞对齐时钟在同一批原始读数上的留出炉等权 MAE 对照（面板 A：$\text{CO}_2$；面板 B：$\text{CO}$）}
\label{fig:q2_clock_bars}
\end{figure}

\begin{table}[htbp]
\centering
\caption{四种对齐时钟在同一批原始分析仪观测点上的留出炉预测指标对比}
\label{tab:q2_clock_comp}
\small\setlength{\tabcolsep}{3.5pt}
\begin{tabular}{clcccc}
\toprule
通道 & 对齐时钟方案 & 同点 MAE ($\text{m}^3/\text{s}$) & 相对同步变化 & 改善炉数 & 有效覆盖率 \\
\midrule
$\text{CO}_2$ & 零时滞同步时钟（$\tau = 0\text{ s}$） & $3.5115$ & $0.00\%$ & — & $99.54\%$ \\
$\text{CO}_2$ & 保守外折回退时钟（平均提前 $6.85\text{ s}$） & $3.5149$ & $+0.10\%$ & $4/10$ & $98.03\%$ \\
$\text{CO}_2$ & \textbf{高流量事件时滞（$\tau = 24.5\text{ s}$）} & $\bm{3.4000}$ & $\bm{-3.18\%}$ & $\bm{6/10}$ & $\bm{95.75\%}$ \\
$\text{CO}_2$ & \textbf{主吹代表时滞（$\tau = 38.0\text{ s}$）} & $\bm{3.4042}$ & $\bm{-3.06\%}$ & $\bm{6/10}$ & $\bm{93.67\%}$ \\
\midrule
$\text{CO}$ & 零时滞同步时钟（$\tau = 0\text{ s}$） & $15.8295$ & $0.00\%$ & — & $99.54\%$ \\
$\text{CO}$ & 保守外折回退时钟（平均提前 $0.00\text{ s}$） & $15.8295$ & $0.00\%$ & — & $99.54\%$ \\
$\text{CO}$ & \textbf{高流量事件时滞（$\tau = 24.5\text{ s}$）} & $\bm{15.4328}$ & $\bm{-2.51\%}$ & $\bm{9/10}$ & $\bm{95.75\%}$ \\
$\text{CO}$ & \textbf{主吹代表时滞（$\tau = 38.0\text{ s}$）} & $\bm{15.2139}$ & $\bm{-3.89\%}$ & $\bm{10/10}$ & $\bm{93.67\%}$ \\
\bottomrule
\end{tabular}
\end{table}

由图 \ref{fig:q2_optical_lag}--图 \ref{fig:q2_clock_bars} 和表 \ref{tab:q2_clock_comp} 可见，引入正向时滞对齐后，两类气体的留炉预测误差均较零时滞同步基准明显下降：$\text{CO}_2$ 的误差曲线在 $24.5\sim 38.0\text{ s}$ 形成谷底（全局最低点位于 $38.0\text{ s}$，同点 MAE 较同步基准下降超过 $3\%$）；$\text{CO}$ 在 $\tau = 24.5\text{ s}$ 和 $\tau = 38.0\text{ s}$ 下的留炉 MAE 分别降至 $15.4328$ 和 $15.2139\text{ m}^3/\text{s}$（分别在 $9/10$ 和 $10/10$ 个炉次上实现误差改善，最大降幅 $3.89\%$），与前文辨识的 $24.5\sim 39.0\text{ s}$ 时滞范围相互印证。

\textbf{（2）重构方法与边界条件的敏感性检验}

在报告区 $3\,170$ 个共同遮蔽点上，对比本文非负三次 B 样条（$\lambda=0.1$）、分段线性插值与保形分段三次插值（PCHIP）的点值 MAE 与 $5\text{ s}$ 遮蔽区间积分 MAE，结果如图 \ref{fig:q2_recon_comp} 所示。

\begin{figure}[htbp]
\centering
\includegraphics[width=0.92\textwidth]{figures/问题二/图12.png}
\caption{三种烟气重构方法在报告区共同遮蔽点上的点值 MAE（面板 A）与区间积分 MAE（面板 B）对比}
\label{fig:q2_recon_comp}
\end{figure}

由图 \ref{fig:q2_recon_comp} 可见，三种方法在遮蔽点上的误差均处于极低水平（非负三次 B 样条的 $\text{CO}_2$ 与 $\text{CO}$ 点值 MAE 仅为 $0.0581$ 和 $0.1512\text{ m}^3/\text{s}$，$5\text{ s}$ 块积分 MAE 仅为 $0.2333$ 和 $0.6037\text{ m}^3$，块积分有符号偏差仅 $-0.0220$ 和 $-0.0123\text{ m}^3$，且点值与积分精度均优于 PCHIP；分段线性插值虽适合两侧紧邻锚点的短缺口局部补值，但导数不连续、无法平滑测量毛刺）。此外，对累计氧表施加段末体积硬约束后，报告区 $3\,230$ 个遮蔽点的整体 MAE 从 $2.4317\text{ m}^3$ 变为 $2.4316\text{ m}^3$，内部精度几乎不变；将附件 5 和附件 7 断点后的极低值尾段保留或将截断边界平移 $\pm 1$ 个更新周期，两炉累计排放总量的变化幅度均小于 $0.28\%$，证明重构结果对边界条件高度稳健。


%参考文献
\begin{thebibliography}{9}%宽度9
\bibitem{1} Author, Title, Place of Publication: Press, Year of publication.
\end{thebibliography}

\newpage
%附录
\section{附录}

\end{document}


