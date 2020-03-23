モダンな機能を生かして
##########################

モジュールの使い方
==================================================

モジュールの３つの使用方法
--------------------------------------------------

モジュールは Fortran90 の新しい機能のうち、最も重要な機能の１つである。 しかし、これを乱用すると分かりにくいプログラムになってしまう。（例えば、モジュールの共有変数の値は、そのモジュール外のいろいろなサブルーチンから 自由に変更可能であるが、その自由を認めると、その共有変数がどこでどういうタイミングで変更されるのか理解が難しいプログラムになってしまう。）
　そこで、プログラムを分かりやすいものにするため、モジュールの使用方法を以下の３つの類型に限定する。

* 定数参照型モジュール
  定数を参照するためのモジュール。

* 変数参照型モジュール
  変数を参照するためのモジュール。common文をこれに置き換える。

* パッケージ型モジュール
  パッケージを扱うためのモジュールである。（パッケージとは、ある機能を持つサブルーチン群のこと。例えば高速フーリエ変換や、気象モデルにおける大気放射など。）共有変数・初期化サブルーチン・実行サブルーチン・局所サブルーチンからなる。

定数参照型モジュール
-----------------------------

いろいろなサブルーチンで使用する定数を、定数参照型モジュールにまとめる。include と異なり、use prm, only : imaxなどと使用したいパラメータだけuseできる。また、この場合、prmにimaxの宣言がなければコンパイル時にエラーが発生してくれる。

* 定数参照型モジュールの例１　（prm_resol.F90）

.. code-block:: fortran

  module prm_resol
  !
  ! 配列の大きさ(解像度)等を表す変数を設定
  !
    implicit none
    integer,parameter :: imax = 128
    integer,parameter :: jmax = 64
    ( 中略 )
  end module prm_resol

* 定数参照型モジュールの例２　（prm_phconst.F90）

.. code-block:: fortran

  module prm_phconst
  !
  ! 物理定数を表す変数を設定する
  !
    implicit none
    real(8),parameter :: cp   = 1004.6D0   ! 比熱
    real(8),parameter :: grav = 9.80665D0  ! 重力定数
    real(8),parameter :: gasr = 287.04D0   ! 気体定数
    ( 中略 )
  end module prm_phconst

* 上記定数参照型モジュールを参照等をするメインプログラムの例（main.F90）

.. code-block:: fortran

  program main
  !
  !  温度・高度データを標準入力して、エネルギーを標準出力する
  !
    use prm_resol  , only : imax           ! 使うものだけをあげよう
    use prm_phconst, only : cp , grav
    implicit none
  !
    real(8) :: energy     (imax)               
    real(8) :: height     (imax)               
    real(8) :: tmperature (imax)
    integer :: i
  !
    read (5,*) tmperature
    read (5,*) height
    do i=1,imax
      energy(i) = cp * tmperature(i) + grav * height(i)
    enddo
    write(6,*) energy
  end program main

　定数参照型モジュール内の定数の値は、コンパイル時に読み込まれるので、コンパイラが最適化を行いやすくなる。実行時に値を変更することは、もちろんできない。

変数参照型モジュール
-----------------------------

いろいろなサブルーチンで使用する変数を、変数参照型モジュールにまとめることができる。

　変数参照型モジュール内の共有変数の値の設定・変更は、同じモジュール内のサブルーチンでしか行ってはいけない。外部からは値を参照するのみにする。

* 変数参照型モジュールの例　（com_tetens.F90）

.. code-block:: fortran

  module com_tetens
    implicit none
    real(8), save :: table(25000)          ! 共有変数
  !
  contains
  !
  subroutine com_tetens__ini               ! 初期化サブルーチン
          : （table(25000)の値の設定を行う）
  end subroutine com_tetens__ini
  !
  end module com_tetens

* 上記変数参照型モジュールを参照等をするメインプログラムの例　（main.F90）

.. code-block:: fortran

  program main
    use com_tetens,only: &
    & com_tetens__ini,  &                 ! サブルーチンcom_tetens__iniを使用。
    & table                               ! 変数tableを使用。
          :
    call com_tetens__ini                   ! tableの値の設定
          :   （以下、table(25000)の値を参照可能。変更はしない（ルール）。）
          :
  end program main

　変数参照型モジュールの共有変数は、プログラム実行時に最初に一度だけ値を設定し、それ以後変更しないのが理想である。値の変更がある変数については、変数参照型モジュールではなく、引数でサブルーチンに渡す方がプログラムが分かりやすくなる場合が多いであろう。

パッケージ型モジュール
-----------------------------

　パッケージ型モジュール内の共有変数や局所サブルーチンにはprivate 属性をつけ、モジュール内でのみ使用し外部から直接使用しないようにする。外部とのやりとりは、初期化サブルーチンと実行サブルーチンだけを通じて行う。

　初期化サブルーチンで共有変数の初期値の設定を行う。

　実行サブルーチンはパッケージ型モジュール内に１個もしくは複数個あり、パ ッケージ型モジュールのメインの部分である。引数を通じて外部からデータを入力し、計算を行い、引数を通じて外部に必要なデータを出力する。

* パッケージ型モジュールの例　（package1.F90）

.. code-block:: fortran

  module package1
    use prm_resol, only : imax,jmax
    implicit none
    private                                ! private属性をdefaultに
    public :: package1__ini, package1__run ! 外部から使用可
    real(8), save :: kyouyuu(imax,jmax)    ! 共有変数、外部から使用不可
  !
  contains
  !
  subroutine package1__ini(a)              ! 初期化サブルーチン
    real(8), intent(in) :: a(imax,jmax)
          :  （kyouyuu(imax,jmax)に初期値を与える）
          :  （データの入力は、例えば、引数から・NAMELISTから・ファイルから）
          :
  end subroutine package1__ini
  !
  subroutine package1__run(b,c)            ! 実行サブルーチン
    real(8), intent(in)  :: b(imax,jmax)
    real(8), intent(out) :: c(imax,jmax)
          :
    call sub1
          :  （kyouyuu,bの値から、cの値を計算する。）
          :  （kyouyuuの値の変更も可。）
  end subroutine package1__run
  !
  subroutine sub1                          ! 局所サブルーチン
          :　　（モジュール内部でのみ使用）
  end subroutine sub1
  !
  end module package1

　パッケージ型モジュール内でのみ使用する予報変数をモジュール内の共有変数にし、外部から隠蔽する。パッケージ型モジュール内でのみ使用するサブルーチンを局所サブルーチンにし、これも外部から隠蔽する。初期化サブルーチンと実行サブルーチンのインターフェースのみを外部に公開する。

　このようにして個々のモジュールの独立性を高めることにより、プログラム全体の構造が分かりやすくなり、多人数で大規模なプログラムの共同開発を行うことが容易になる。

モジュールの階層構造
-----------------------------

　上の階層のモジュールが下の階層のモジュールをuseし使用する形になる。コンパイル時には、下の階層のモジュールから順にコンパイルしなければいけない。
　サブルーチンはモジュールに属するようにする。サブルーチンの属するモジュールをuseしてからでないとそのサブルーチンを使用できなくなるが、コンパイラがコンパイル時に引数の型が一致しているかどうかチェックを行ってくれる というメリットがある。これによりデバッグが容易になる。

動的割り付け
-----------------------------

　サブルーチン内のワーク的変数を動的にメモリに割り付けることにより、プログラム作成者はワーク的変数の管理から開放され、メモリを有効利用することができる。また実行時に配列の大きさを変更できるので、解像度に依存しないプログラムが作成できる。

* 動的割り付けサブルーチンの例　（sub.F90）

.. code-block:: fortran

  subroutine sub( imax, jmax, a )
    implicit none
    integer, intent(in)    :: imax
    integer, intent(in)    :: jmax
    real(8), intent(inout) :: a(imax,jmax)
  !
    real(8) :: work1(imax,jmax)            ! 自動配列
    real(8), allocatable :: work2(:,:)     ! allocate可能な配列
  !
    allocate(work2(imax,jmax))             ! メモリに割り付ける。
          :  （aの値を計算する。）
          :
    deallocate(work2)                      ! メモリを解放する。
  end subroutine sub

　上の例で、自動配列は自動的にサブルーチンの初めにメモリに割り付けられ、サブルーチンの終わりにメモリを解放する。allocatable 配列では、allocate 文、deallocate文で明示的に行う。

　自動配列はメモリのスタック領域を使用する場合が多く、ヒープ領域を使用する場合が多いallocatable配列より自動配列の方が割付を高速に行えることが多い。しかし、使用可能なスタックのサイズに制限がある場合があり、自動配列で使用できるメモリが制限される場合がある。（ＯＳ・コンパイラによる。）

