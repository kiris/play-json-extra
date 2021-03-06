Play-Json extensions
==========================

> This is a fork of the awesome [Play JSON Extensions](https://github.com/cvogt/play-json-extensions) using the other feature of play.json.extra
> and working for both Scala and ScalaJS
 

### De-/Serialize case classes of arbitrary size (23+ fields allowed)

    case class Foo(
      _1:Int,_2:Int,_3:Int,_4:Int,_5:Int,
      _21:Int,_22:Int,_23:Int,_24:Int,_25:Int,
      _31:Int,_32:Int,_33:Int,_34:Int,_35:Int,
      _41:Int,_42:Int,_43:Int,_44:Int,_45:Int,
      _51:Int,_52:Int,_53:Int,_54:Int,_55:Int
    )

    val foo = Foo(1,2,3,4,5,1,2,3,4,5,1,2,3,4,5,1,2,3,4,5,1,2,3,4,5)
    

#### Create explicit formatter
    import play.json.extra.Jsonx
    implicit def jsonFormat = Jsonx.formatCaseClass[Foo]

    // if your case class uses Option make sure you import
    // one of the below implicit Option Reads to avoid
    // "could not find implicit value for parameter helper: play.json.extra.OptionValidationDispatcher"

    // note: formatCaseClass catches IllegalArgumentException and turns them into JsError enclosing the stack trace as the message
    // this allows using require(...) in class constructors and still get JsErrors out of serialization

#### Then use ordinary play-json
    val json = Json.toJson( foo )
    assert(foo == json.as[Foo])

#### De-/Serialize tuples
    import play.json.extra.tuples._
    val json = Json.parse("""[1,1.0,"Test"]""")
    val res = Json.fromJson[(Int,Double,String)](json)
    assert(JsSuccess((1,1.0,"Test")) === res)

#### De-/Serialize single value classes
    case class Foo(i: Int)
    val json = Json.parse("1")
    val res = Json.fromJson[Foo](json)
    assert(JsSuccess(Foo(1)) === res)

### Option for play-json 2.4

#### implicit Option Reads
    import play.json.extra.implicits.optionWithNull // play 2.4 suggested behavior
    // or
    import play.json.extra.implicits.optionNoError // play 2.3 behavior

#### automatic option validation: `validateAuto`
    val json = (Json.parse("""{}""") \ "s")
    json.validateAuto[Option[String]] == JsResult(None) // works as expected correctly

    // play-json built-ins
    json.validate[Option[String]] // JsError: "'s' is undefined on object: {}"
    json.validateOpt[String] == JsResult(None) // manual alternative (provided here, built-into play-json >= 2.4.2)
    
#### automatic formatting of sealed traits, delegating to formatters of the subclasses
#### formatSealed uses orElse of subclass Reads in random order, careful in case of ambiguities of field-class correspondances
    sealed trait SomeAdt
    case object A extends SomeAdt
    final case class X(i: Int, s: String) extends SomeAdt
    object X{
      implicit def jsonFormat: Format[X] = Jsonx.formatCaseClass[X]
    }
    object SomeAdt{
      import SingletonEncoder.simpleName         // required for formatSingleton
      import play.json.extra.formatSingleton // required if trait has object children
      implicit def jsonFormat: Format[SomeAdt] = Jsonx.formatSealed[SomeAdt]
    }

    Json.parse("""A""").as[SomeAdt] == A
    Json.parse("""{"i": 5, "s":"foo", "type": "X"}""").as[SomeAdt] == X(5,"foo")

### experimental features (will change)
#### Serialization nirvana - formatAuto FULLY automatic de-serializer (note: needs more optimized internal implementation)

    sealed trait SomeAdt
    case object A extends SomeAdt
    final case class X(i: Int, s: String) extends SomeAdt
    object Baz
    case class Bar(a: Int, b:Float, foo: Baz.type, o: Option[Int])
    case class Foo(_1:Bar,_11:SomeAdt, _2:String,_3:Int,_4:Int,_5:Int,_21:Int,_22:Int,_23:Int,_24:Int,_25:Int,_31:Int,_32:Int,_33:Int,_34:Int,_35:Int,_41:Int,_42:Int,_43:Int,_44:Int,_45:Int,_51:Int,_52:Int,_53:Int,_54:Int,_55:Int)
    val foo = Foo(Bar(5,1.0f, Baz, Some(4): Option[Int]),A,"sdf",3,4,5,1,2,3,4,5,1,2,3,4,5,1,2,3,4,5,1,2,3,4,5)
    val foo2 = Foo(Bar(5,1.0f, Baz, None: Option[Int]),X(5,"x"),"sdf",3,4,5,1,2,3,4,5,1,2,3,4,5,1,2,3,4,5,1,2,3,4,5)
    
    import play.json.extra.implicits.optionWithNull
    val fmt2: Format[Foo] = Jsonx.formatAuto[Foo] // not implicit to avoid infinite recursion

    {
      implicit def fmt3: Format[Foo] = fmt2    
      val json = Json.toJson( foo )
      assert(foo === json.as[Foo])
      val json2 = Json.toJson( foo2 )
      assert(foo2 === json2.as[Foo])
    }