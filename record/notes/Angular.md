# Angular学习

## 入门基础

+ 安装Angular CLI: `npm install -g @angular/cli`.
+ 创建一个Angular项目: `ng new PROJECT_NAME`.
+ 运行Angular项目: `ng serve --open`.
+ 新增模块:`ng generate component COMPONENT_NAME`.
+ 双向绑定在input中使用`[(ngModel)]`, 需要在`app.module.ts`中导入`import { FormsModule } from '@angular/forms';` -> `imports: [BrowserModule,FormsModule]`.
+ FOR循环: `<li *ngFor="let ONE of MANY">`.
+ IF判断: `<div *ngIf="REQUIREMENT"> DO SOMETHING</div>`.
+ Angular信息从父模块传递给子模块.使用属性绑定进行信息传递.
  - 子模块需要对特定的属性(需要父模块传递的)进行特殊声明: `@Input()`.(注需要从头部导入进来.)
  - 在父模块中使用时: `<SON_COMPONENT [SON_COMPONENT_PROPERTY_NAME]="FATHER_COMPONENT_PROPERTY_NAME"></SON_COMPONENT>`.
+ 生成新的Service: `ng generate service SERVICE_NAME`.
+ 在模块中注入Service, 首先在头里导入服务, 然后在构造函数中手动注入: `constructor(private SERVICE_ALIAS: SERVICE_NAME)`, 然后在内部通过`this`访问. (注: private/public区别: private不能在html中显式使用).
+ 对于异步处理的数据, 可以使用`Observable`包装对象,(注构造模拟的Observable数据可以使用`of`进行包装), 然后在使用时进行`subscribe`进行异步订阅处理.
+ 添加动态路由:
+ `ng generate module app-routing --module=app`.(注`--module=app`自动在`AppModule`中进行注册.)
+ 定义一个路由链接: `const routes: Routes = [ { path: 'PATH_NAME', component: COMPONENT_NAME }];`.
+ 导入路由: `imports: [ RouterModule.forRoot(routes) ]`.
+ 在需要使用路由的地方: `<router-outlet></router-outlet>`.
+ 使用路由跳转: `<a routerLink="/PATH_NAME">LINK_NAME</a>`.
+ 添加默认路由: `{ path: '', redirectTo: '/PATH_NAME', pathMatch: 'full' },`.
+ 使用路由传递参数: `{ path: 'PATH_NAME/:PARAMETER_NAME', component: COMPONENT_NAME },`.
+ 获取路由参数:
  + 导入`ActivatedRoute`: `constructor(private route: ActivatedRoute)`.
  + 获取对应参数: `this.route.snapshot.paramMap.get('PARAMETER_NAME')`.