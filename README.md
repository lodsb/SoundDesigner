*************
## How to build:

###First,you need:  
A running jdk install and sbt (http://www.scala-sbt.org/) 

###Second, dependencies: 
all dependencies are included in the supplied libraries directory, the only part of the framework missing is 
reakt (https://github.com/lodsb/reakt), you have to build reakt first.

1. check out reakt from github 
	git clone http://www.scala-sbt.org/
2. build it
	sbt compile
3. publish it on your system, so it is packaged for UltraCom
	sbt publish-local

###Finally:
There is not much more to do:
1. check out UltraCom from github:
	git clone https://github.com/lodsb/UltraCom.git
2. build it
	sbt compile
3. publish it (so you can build the examples)
	sbt publish local

## Example Projects
In examples/ are small project files, you can build/execute them by running sbt compile/run in the corresponding directory

## Example
This lenghty example is taken from examples/tutorial_one and shows how the API looks like and how it can be used


object TutorialOne extends Application {
	/*
			Settings, such as <b>the application name<b>, display properties, etc are set in Settings.txt
	 */

	def main(args: Array[String]): Unit = {
		this.execute(false)
	}

	override def startUp() = {
		val scene = new TutorialOneScene(this, "Scene of Tutorial One");
		this.addScene(scene)
	}

}


class TutorialOneScene(app: Application, name: String) extends Scene(app,name) {

	// Show touches
	showTracer(true)

	/*
		Basics

		- component creation
		- changing properties
	 */

	// Create two UI components ...
	var textField = TextArea();
	var slider = Slider(0,100);

	// ^^ this can also be done java style using plain mt4j
	var textField2 = new MTTextArea(app)
	var slider2 = new MTSlider(app,100,100,100,20,0,100);

	//Adding the textarea reakt style to canvas
	canvas += textField;

	//Adding slider to canvas, original java style
	this.getCanvas.addChild(slider);

	//Adding arrays as children to an object (could be canvas), could be single object, too
	textField += textField2++slider2

	//Removing a child
	textField -= textField2

	// Setting position ... java style from original MT4J
	textField.setPositionGlobal(new Vector3D(app.width / 2f, app.height / 2f));

	// same, but reakt style (makes use of properties, see below)
	textField.globalPosition := Vec3d(app.width / 2f, app.height / 2f);

	// setting and reading Properties
	textField.text:="foo"
	textField.text() ="foo"
	val foo = textField.text();
	slider.globalPosition() = Vec3d(100,200);

	/*
		other component properties are
		- global and relative position
		- rotation
		- padding ...

	 */


	/*
		Linking/Connecting event streams

		Some examples about making use of event streams
		- linking
		- modifying
		- general observing

		NOTE: All event streams are thread-save! so no worries about cross-connecting UI/Network/Input streams :)

	 */

	// Connect a slider to a textarea (show slider values)
	// - every time the slider outputs a value, it is updating the text field
	// - the type conversion of Float to String is done automatically (mostly works in simple cases)
	textField.text <~ slider.value + ""

	// convert the Float event stream to a Rotation and a Vector Event stream
	textField.localRotation <~ slider.value.map({ x => Rotation(degreeX = x*0.4f) })
	textField.globalPosition <~ slider.value.map({ x => Vec3d(x*10f,x*10f,0) })

	// convert the Float event stream to a Color event stream
	textField.strokeColor <~ slider.value.map({ x => Color(0f,x,255f-x) })
	// 						              ^^ this and the examples before create other anonymous signals,
	// 										 we can't simply disconnect them explictly
	// so the only solution is to disconnect all sources (from strokeColor in this case)
	textField.strokeColor.disconnectAll

	// but we can also use a variable for the intermediate signal, so connecting/disconnecting can be done this way
	val intermediateSignal = slider.value.map({ x => Color(0f,x,255f-x) })
	textField.strokeColor <~ intermediateSignal
	textField.strokeColor.disconnect(intermediateSignal)
	// or with an overloaded operator
	textField.strokeColor |~ (intermediateSignal)

	// reconnect...
	textField.strokeColor <~ intermediateSignal

	/* <b>Other signal operators are:</b>
	   ~> 				: connect; works to successively concatenate signals as well!
	   signal() = value : push value into signal
	   :> merge         : merge two signals to one - creates tuple signal

	   arithmetic
	   +, - , /, * 		- creates a signal that contains the result of two signals' arithmetic operation

	   min, max 		- creates signal containing the min/max of two signals

	   comparisons 		- creates a boolean signal
	   < , > , <= , >=, eq

	   DIY:
	   binop(fun: (T,T) => T) - roll your binary op
	*/


	/*
		(little) Finite State Machine and Buttons

		An utterly useless example
	 */
	val button = Button("Some text...");
	val slider3= Slider(0,20)

	button.globalPosition() = Vec3d(100,300)
	slider3.globalPosition() = Vec3d(100,500)

	canvas += button

	// just for fun: create a sequence of lines with a random start point
	val r = new Random()
	def rval(): Float = { (r.nextFloat()*600)%300}

	val lines = (1 to 25).map({x =>
		val l = Line()
		l.startPosition() = Vec3d(rval()*2,rval()*2,rval())
		l.setStrokeColor( Color(rval(),rval(),rval()) )
		// connect the end position of each line to the (position of the) textfield
		l.endPosition <~ textField.globalPosition
		l
	})

	val myFSM = new StateMachine { // Derive & define a StateMachine

		fsm {
			// S - is used to define the start state MyStart - the "'" is used by scala to denote symbols
			S('MyStart)

			// define the state MyStart
			'MyStart is {
				println("I am in state MyStart")

				button.text() = "Trigger me!"

				// tell the state to react on input,
				// this can use the usual pattern matching, e.g. tuples, types,...
				react {

					// do something if true is received:
					// transition to state ShowMoreUI, before that
					// show all lines, we created before
					case true => {
						// add sequence (array) to the canvas
						canvas += lines
						->('UseMoreUI) // with UTF symbol: →('ShowMoreUI)
					}
				}
			}

			'UseMoreUI is {
				println("I am in state UseMoreUI")
				button.text() = "Use sliders?"

				canvas += slider3

				react {
					case true => {
						canvas -= lines ++ slider3
						->('MyStart)
					}

					case x:Float => {
						slider.fillColor() = Color(x,0,0)
						->('UseMoreUI)
					}

					case x:String => {
						textField.text() = x+" use button to go back"
						->('UseMoreUI)
					}

				}
			}
		}
	}

	// push the various UI outputs to the state machine and let it decide how to react
	button.pressed.observe({ x => myFSM.consume(x); true })
	slider.value.observe({ x => myFSM.consume(x); true })
	slider3.value.observe({ x => myFSM.consume("Some val "+x); true })
}

