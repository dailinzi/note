## 运行环境
```
Microsoft.EntityFrameworkCore.Tools
Microsoft.EntityFrameworkCore.SqlServer
```
### 方式1
##### 基于现有数据库创建模型即dbffirst
包管理器运行：
```
dotnet ef dbcontext scaffold “Server=localhost;Database=efcorelearn;User=root;Password=password;” Microsoft.EntityFrameworkCore.SqlServer” -OutputDir Models
```
*  Server=你的数据所在的ip地址，
*  Database=你的数据库名称
*  User 是你连接数据的用户名，
* Password=你的Password，
* -O 后的Models是指文件名称。

###### 在Models文件夹里生成的两个类，最重要的是 efcorelearnContext和实体类Student; 其中efcorelearnContext 是承上启下的关键,它是对ADO.Net的封装

###### OnModelCreating方法，是用来将实体与数据库表映射的；DbSet  数据库的每个表在这个类中都是以DbSet 的属性出现。

### CRUD体验
##### 新增 add +  saveChanges（）
```
		static void Main(string[] args)
        {
            using (var db=new efcorelearnContext())
            {
                Student student = new Student()
                {
                    Name = "小明",
                    Class = "软件一班"
                };
                db.Students.Add(student);
                db.SaveChanges();//提交更改后才能保存到数据库。
            }
```

##### 修改 update /  db.Entry(实体对象).State = EntityState.Modified /EntityState ss = db.Entry(s实体对象).State;  / db.Entry(stu3).State = EntityState.; /db.Students.Attach(stu3); db.Entry(stu3).State = EntityState.Modified;

```
   using (var db=new efcorelearnContext())
            {
                //更新方式1 Update
                Student student = new Student()
                {
                    StudentId = 1,
                    Name = "小明",
                    Class = "软件2班"
                };
                db.Students.Update(student);
                db.SaveChanges();

                //更新方式2 先查询，再修改

                var stu1 = db.Students.AsEnumerable().First(a => a.StudentId == 1);//或者var stu1=db.Students.Single(a => a.StudentId == 1)
                stu1.Name = "小红";
                EntityState ss = db.Entry(stu1).State;//unchanged
                db.SaveChanges();

                //更新方式3 设置 EntityState的状态和Update类似
                Student stu2 = new Student();
                stu2.Name = "小李";
                stu2.StudentId = 1;
                stu2.Class = "软件3班";
                db.Entry(stu2).State = EntityState.Modified;
                db.SaveChanges();

                //更新方式4 Attach方法
                Student stu3 = new Student();
                stu3.Name = "小明";
                stu3.StudentId = 1;
                stu3.Class = "软件1班";
                db.Students.Attach(stu3);
                db.Entry(stu3).State = EntityState.Modified;
                db.SaveChanges();
            }
```



### 方式2 CodeFirst
* 创建 DbContext 实例 根据上下文跟踪实体实例
* 创建 实体模型，在DbContext 实例类中添加DbSet<实体>属性
* 根据业务需求进行增删改查， 调用 SaveChanges 或 SaveChangesAsync，将更改应用到数据库

##### OnConfiguring
```
 protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            if (!optionsBuilder.IsConfigured)
            {

                ooptionsBuilder.UseSqlServer("Server=127.0.0.1;User Id=sa;Password=ludailin;Database=EFCoreStuMes");
                optionsBuilder.LogTo(Console.WriteLine);//日志输出到控制台
               // optionsBuilder.AddInterceptors(new SoftDeleteInterception()); //添加拦截器软删除
                optionsBuilder.EnableThreadSafetyChecks(false);//关闭并安全检测

            }
        }
       }

```
##### OnModelCreating
模型配置有两种方式
* FluentAPI
* 数据注解
```
  protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.UseCollation("utf8_general_ci")
                .HasCharSet("utf8");

            modelBuilder.Entity<Student>(entity =>
            {
                entity.Property(e => e.Class).HasMaxLength(50);

                entity.Property(e => e.Name).HasMaxLength(50);
            });

        }
```

映射关系配置 学生 地址 课程 老师
```
public class EFLearnDbContext: DbContext
    {
        public EFLearnDbContext()
        {
        }
        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
          
           optionsBuilder.UseMySql("server=192.168.0.106;database=EFCoreLearn;user=root;password=123456", Microsoft.EntityFrameworkCore.ServerVersion.Parse("8.0.28-mysql"));
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {

            #region 基本配置
            modelBuilder.Entity<Student>().Property(x => x.StudentId).ValueGeneratedOnAdd();//设置Id自增
            //设置姓名最大长度为50，字符为unicode,不能为空
            modelBuilder.Entity<Student>().Property(x => x.Name).HasMaxLength(50).IsUnicode().IsRequired();
            //设置性别最大长度为5 字符为Unicode，不能为空
            modelBuilder.Entity<Student>().Property(x => x.Sex).HasMaxLength(5).IsUnicode().IsRequired();
            //一对一只需要配置一个类就行了，
            modelBuilder.Entity<Student>().HasOne(x => x.Address).WithOne(x => x.Student).HasForeignKey<StudentAddress>(ad=>ad.StudentId);
            #endregion

            #region  Course
            modelBuilder.Entity<Course>().Property(x => x.Name).HasMaxLength(50).IsUnicode();
            //课程一对一
            modelBuilder.Entity<Course>().HasOne(x => x.Teacher).WithMany(t => t.Courses);
            modelBuilder.Entity<Course>().HasMany(x => x.Students).WithMany(st => st.Courses);
            #endregion

            #region Teacher配置
            modelBuilder.Entity<Teacher>().Property(x => x.Name).HasMaxLength(50).IsUnicode();
            modelBuilder.Entity<Teacher>().Property(x => x.Title).HasMaxLength(50).IsUnicode();
            #endregion

            #region 配置地址
            modelBuilder.Entity<StudentAddress>().Property(x=>x.City).HasMaxLength(100).IsRequired().IsUnicode();
            modelBuilder.Entity<StudentAddress>().Property(x=>x.Address).HasMaxLength(500).IsRequired().IsUnicode();
            #endregion
            
        }


        public DbSet<Student> Students { get; set; }
        public DbSet<Teacher> Teachers { get; set; }
        public DbSet<Course> Courses { get; set; }
        public DbSet<StudentAddress> Addresses { get; set; }
    }
```

批量添加  AddRange
```
  //1.添加一些数据

    Course course1 = new Course() { CourseId = 3001, Name = "高等数学" };
    Course course2 = new Course() { CourseId = 3002, Name = "计算机原理" };
    Course course3 = new Course() { CourseId = 3003, Name = "操作系统原理" };
    Course course4 = new Course() { CourseId = 3004, Name = "编译原理" };

  
    Teacher teacher1 = new Teacher() { TeacherId = 10001, Name = "张教授", Title = "教授" };
    Teacher teacher2 = new Teacher() { TeacherId = 10002, Name = "王讲师", Title = "讲师" };
   
    teacher1.Courses=new Course[] { course1,course2 };
    teacher2.Courses = new Course[] { course3, course4 };

    dbContext.Teachers.AddRange(teacher1, teacher2);

    StudentAddress Address1 = new StudentAddress()
    {
        StudentAddressId = 37001,
        Address = "北京朝阳区",
        City = "北京",
    };

    StudentAddress Address2 = new StudentAddress()
    {
        StudentAddressId = 37002,
        Address = "上海徐汇区",
        City = "上海",
    };

    StudentAddress Address3 = new StudentAddress()
        {
        StudentAddressId = 37003,
            Address = "广州白云区",
            City = "广州",
        };

   

    Student student1 = new Student()
    {
        StudentId = 2022001,
        Name = "王二小",
        Age = 19,
        Sex="男",
        Address =Address1,
        Courses = new Course[] {course1, course2, course3 },

    };

    Student student2 = new Student()
    {
        StudentId = 2022002,
        Name = "张小五",
        Age = 20,
        Sex = "男",
        Address = Address2,
        Courses=new Course[] { course1,course2 },
    };

    Student student3 = new Student()
    {
        StudentId = 2022003,
        Name = "刘小花",
        Age = 20,
        Sex="女",
        Address = Address3,
        Courses = new Course[] { course1, course3 },

    };
    dbContext.Students.AddRange(student1, student2,student3);
    
    dbContext.SaveChanges();
```
包含导航属性查询的时候，可以使用Include方法进行相关的导航属性
```
 var students= dbContext.Students.Include(x=>x.Address).ToList();

    foreach (var st in students)
    {
        Console.WriteLine($"StudentId:{st.StudentId},Name:{st.Name},City:{st.Address.City}, Address:{st.Address.Address}");
    }
```
学生删除后，发现相关的地址信息，以及选择的课程信息对照表也被删除了，这是由于在设置的EFCore默认了删除的行为为级联的方式。